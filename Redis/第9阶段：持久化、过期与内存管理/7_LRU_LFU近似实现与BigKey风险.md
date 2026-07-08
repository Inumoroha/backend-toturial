# 7. LRU、LFU 近似实现与 Big Key 风险

上一节学习了内存淘汰策略。

这一节进一步理解两个生产排查中非常重要的话题：

- Redis 的 LRU/LFU 是近似实现。
- Big Key 会带来内存、网络和阻塞问题。

学完这一节后，你应该能够：

- 理解 Redis LRU 不是严格 LRU。
- 理解 LFU 关注访问频率。
- 知道如何发现 Big Key。
- 知道 Big Key 为什么会阻塞。
- 使用 `UNLINK` 降低删除阻塞风险。

---

## 一、什么是 LRU

LRU：

```text
Least Recently Used，最近最少使用。
```

直觉：

```text
越久没被访问的 key，越应该被淘汰。
```

如果 Redis 做缓存，LRU 很符合直觉：

```text
最近访问过的数据更可能再次被访问。
```

---

## 二、Redis 为什么不用严格 LRU

严格 LRU 通常需要维护全局链表或复杂结构。

每次访问 key 都要更新位置。

这会增加：

- 内存开销。
- CPU 开销。
- 实现复杂度。

Redis 追求高性能，所以使用近似 LRU。

简单理解：

```text
Redis 随机采样一批 key，
从样本中选择最久未访问的 key 淘汰。
```

---

## 三、采样数量

Redis 有配置：

```conf
maxmemory-samples 5
```

含义：

```text
淘汰时采样 5 个 key，从中选一个更适合淘汰的。
```

采样数量越大：

- 淘汰结果越接近真实 LRU。
- CPU 成本越高。

一般不需要随意调整。

---

## 四、什么是 LFU

LFU：

```text
Least Frequently Used，最不经常使用。
```

它关注访问频率。

适合：

- 长期热点明显。
- 某些 key 经常被访问。
- 希望保留高频热点数据。

例如：

```text
热门商品详情
热门文章详情
首页配置
```

这些 key 即使短时间没访问，也可能仍然重要。

---

## 五、LRU 和 LFU 怎么选

| 访问模式 | 推荐 |
| --- | --- |
| 最近访问更能代表未来访问 | LRU |
| 长期热点更明显 | LFU |
| 访问模式不清楚 | 先用 LRU，观察监控 |

示例：

```text
新闻详情缓存：LRU 可能更合适
热门商品缓存：LFU 可能更合适
```

但实际要看业务访问数据。

不要只凭感觉。

---

## 六、什么是 Big Key

Big Key 没有绝对统一标准。

可以从两个角度看：

```text
value 很大
元素很多
```

例如：

- String 超过几 MB。
- Hash 有几十万 field。
- List 有几十万元素。
- Set 有几十万 member。
- ZSet 有几十万 member。

具体阈值要结合业务、网络和 Redis 规模。

---

## 七、Big Key 为什么危险

Big Key 会带来：

- 内存占用大。
- 读取传输慢。
- 序列化和反序列化慢。
- 删除可能阻塞。
- 迁移和复制成本高。
- AOF/RDB 处理压力大。
- Cluster 下槽位迁移困难。

一个 20MB 的 String，即使 `GET` 是单条命令，也会占用网络和客户端处理时间。

一个 100 万元素的 Set，删除时可能阻塞 Redis。

---

## 八、发现 Big Key

命令：

```redis
MEMORY USAGE key
```

查看 key 估算内存。

扫描时不要用：

```redis
KEYS *
```

可以用：

```redis
SCAN 0 COUNT 100
```

Redis 自带工具也可以：

```bash
redis-cli --bigkeys
```

它会扫描并报告各类型最大 key。

生产中运行扫描要控制时间和频率。

---

## 九、查看集合大小

不同类型：

```redis
STRLEN string:key
HLEN hash:key
LLEN list:key
SCARD set:key
ZCARD zset:key
```

这些命令可以帮助判断元素数量。

但元素数量不等于实际内存。

实际还要看：

```redis
MEMORY USAGE key
```

---

## 十、删除 Big Key 的风险

普通删除：

```redis
DEL big:set
```

如果 key 很大，释放内存可能耗时较长，阻塞 Redis 主线程。

更推荐：

```redis
UNLINK big:set
```

`UNLINK` 会把 key 从 keyspace 中移除，后续内存释放交给后台线程处理。

它能降低删除大 key 的阻塞风险。

---

## 十一、Big Key 如何治理

治理思路：

### 1. 拆分

例如一个大 Hash：

```text
user:events
```

拆成：

```text
user:events:20260705
user:events:20260706
```

### 2. 分片

```text
set:active_users:shard:0
set:active_users:shard:1
```

### 3. 控制 value 大小

不要把巨大 JSON 整体塞 Redis。

### 4. 设置 TTL

临时数据要有生命周期。

### 5. 使用合适结构

只要计数就用 HyperLogLog，不要保存完整 Set。

---

## 十二、Hot Key 和 Big Key 不一样

Big Key：

```text
key 很大。
```

Hot Key：

```text
key 访问很热。
```

一个 key 可以：

- 只是 Big Key，不热。
- 只是 Hot Key，不大。
- 又大又热，最危险。

例如：

```text
热门首页配置 5MB，所有请求都读。
```

这会同时带来网络、CPU 和 Redis 压力。

---

## 十三、Go 里避免大 value

常见错误：

```go
// 把完整文章列表缓存成一个巨大 JSON
rdb.Set(ctx, "home:feed", hugeJSON, time.Hour)
```

更合理：

```text
home:feed:ids -> 只缓存 ID 列表
article:detail:{id} -> 单篇文章详情
```

这样：

- 单个 value 更小。
- 更新影响范围更小。
- 可以按需加载详情。

---

## 十四、常见错误

### 1. 以为 LRU 是严格全局 LRU

Redis 使用近似采样。

### 2. 盲目选择 LFU

LFU 适合长期热点明显的场景，不是永远更好。

### 3. 用 `DEL` 删除巨大 key

可能阻塞 Redis。

### 4. 把大列表缓存成一个 JSON

读取、传输和更新都很重。

### 5. 不区分 Big Key 和 Hot Key

治理方法不同，但都需要监控。

---

## 十五、本节练习

请完成下面练习：

1. 用自己的话解释 LRU。
2. 用自己的话解释 LFU。
3. 查看 `maxmemory-samples` 配置。
4. 用 `MEMORY USAGE` 查看一个 key。
5. 用 `HLEN`、`SCARD` 或 `ZCARD` 查看集合大小。
6. 构造一个大 Set，然后用 `DEL` 和 `UNLINK` 对比思路。
7. 设计一个大 JSON 缓存的拆分方案。
8. 解释 Big Key 和 Hot Key 的区别。

---

## 十六、本节小结

这一节你学习了 LRU、LFU 和 Big Key。

你需要记住：

- Redis 的 LRU/LFU 是近似实现，依赖采样。
- LRU 关注最近访问，LFU 关注访问频率。
- Big Key 会影响内存、网络、删除、复制和迁移。
- 发现 Big Key 可以用 `MEMORY USAGE`、类型长度命令和 `redis-cli --bigkeys`。
- 删除大 key 优先考虑 `UNLINK`。
- 治理 Big Key 的核心是拆分、分片、控制 value 大小和设置生命周期。

下一节我们做第 9 阶段综合实践。

