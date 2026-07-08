# 6. Big Key 与 Hot Key 发现治理

Big Key 和 Hot Key 是 Redis 生产问题中最常见的两个关键词。

Big Key 是“大”，Hot Key 是“热”。它们可能独立出现，也可能叠加成非常危险的问题。

学完这一节后，你应该能够：

- 区分 Big Key 和 Hot Key。
- 发现 Big Key。
- 发现 Hot Key。
- 设计治理方案。
- 知道本地缓存和多级缓存适合什么场景。

---

## 一、Big Key 和 Hot Key 的区别

Big Key：

```text
单个 key 的 value 很大，或元素很多。
```

Hot Key：

```text
单个 key 被极高频访问。
```

危险组合：

```text
一个很大的 key，同时被大量请求访问。
```

它会同时放大：

- Redis CPU。
- 网络带宽。
- 客户端反序列化。
- 延迟抖动。

---

## 二、Big Key 发现方式

命令：

```redis
MEMORY USAGE key
STRLEN string:key
HLEN hash:key
LLEN list:key
SCARD set:key
ZCARD zset:key
```

扫描工具：

```bash
redis-cli --bigkeys
```

注意：

```text
生产扫描要谨慎，控制频率和时间。
```

最好在从节点或低峰期执行。

---

## 三、Big Key 治理

治理方式：

- 拆 key。
- 分片。
- 控制 value 大小。
- 分页读取。
- 避免全量返回。
- 使用 `UNLINK` 删除。
- 设置 TTL。
- 换更合适的数据结构。

示例：

```text
错误：home:feed -> 巨大 JSON
更好：home:feed:ids -> ID 列表
     article:detail:{id} -> 单篇详情
```

---

## 四、Hot Key 发现方式

Hot Key 发现通常来自：

- 应用访问日志。
- Redis 代理统计。
- 客户端埋点。
- 业务经验。
- Redis `--hotkeys`。

命令：

```bash
redis-cli --hotkeys
```

注意：`--hotkeys` 依赖 LFU 相关信息，使用前要了解环境和配置。

更通用的方式是应用侧统计：

```text
按 key 前缀、业务 ID 统计访问次数。
```

---

## 五、Hot Key 的风险

Hot Key 会导致：

- 单个 Redis 节点 CPU 飙高。
- 单 key 请求排队。
- 网络出口压力。
- 某个 Cluster 节点倾斜。
- 后端服务等待 Redis。

在 Cluster 中，Hot Key 所在 slot 的节点会特别忙。

Cluster 分片不能自动拆散单个 Hot Key。

---

## 六、Hot Key 治理方式

常见治理：

### 1. 本地缓存

适合：

- 热点配置。
- 热门文章。
- 首页数据。
- 能接受短暂旧数据。

结构：

```text
应用本地缓存 -> Redis -> DB
```

### 2. 逻辑过期

热点 key 不直接物理过期，过期后先返回旧数据并异步重建。

### 3. key 副本

把一个热点 key 拆成多个副本：

```text
hot:article:1001:0
hot:article:1001:1
hot:article:1001:2
```

读请求随机读一个副本。

### 4. 限流降级

极端流量下保护 Redis 和数据库。

---

## 七、本地缓存注意点

本地缓存优点：

- 减少 Redis 压力。
- 降低网络开销。
- 提升响应速度。

缺点：

- 多实例一致性更复杂。
- 数据可能短暂旧。
- 需要 TTL。
- 需要失效通知或定时刷新。

适合读多写少、可接受旧数据的热点。

不适合权限、余额、支付状态。

---

## 八、多级缓存

结构：

```text
本地缓存 -> Redis -> 数据库
```

读流程：

```text
1. 先查本地缓存。
2. 未命中查 Redis。
3. Redis 未命中查数据库。
4. 回写 Redis 和本地缓存。
```

写流程要考虑：

- 更新数据库。
- 删除 Redis。
- 通知本地缓存失效。
- TTL 兜底。

多级缓存提升性能，也增加一致性复杂度。

---

## 九、热点 key 副本的取舍

key 副本能分散读压力。

但写入和失效要同时处理多个副本。

适合：

- 极高读、低写。
- 数据允许短暂不一致。
- 明确知道热点 key。

不适合：

- 写频繁。
- 强一致。
- key 数量太多且热点变化快。

---

## 十、Big + Hot 的处理

如果一个 key 又大又热：

```text
先拆大，再降热。
```

例如热门商品详情 5MB：

```text
1. 拆详情，只保留必要字段。
2. 图片、描述等放对象存储或独立缓存。
3. 加本地缓存。
4. 使用逻辑过期。
5. 控制接口返回字段。
```

不要直接复制 10 份 5MB 热点 key。

那会把内存问题放大。

---

## 十一、常见错误

### 1. 只治理 Hot，不治理 Big

热点副本会放大大 value 的内存占用。

### 2. 本地缓存没有 TTL

失效通知丢了会长期读旧数据。

### 3. Cluster 中以为分片能解决 Hot Key

单个 key 仍然只在一个 slot。

### 4. 对强一致数据加本地缓存

权限、余额、支付状态不适合随意本地缓存。

### 5. Big Key 删除直接 DEL

删除大 key 优先考虑 `UNLINK`。

---

## 十二、本节练习

请完成下面练习：

1. 用自己的话区分 Big Key 和 Hot Key。
2. 使用 `MEMORY USAGE` 检查一个 key。
3. 使用类型长度命令检查集合大小。
4. 设计一个应用侧 Hot Key 统计方案。
5. 为热门文章详情设计本地缓存策略。
6. 为热点配置设计 Pub/Sub 失效通知。
7. 思考一个又大又热的 key 应该怎么拆。

---

## 十三、本节小结

这一节你学习了 Big Key 和 Hot Key。

你需要记住：

- Big Key 是大，Hot Key 是热。
- Big Key 会带来内存、网络、删除和复制问题。
- Hot Key 会让单节点或单 slot 压力过高。
- 本地缓存、多级缓存、逻辑过期、热点副本都能治理热点。
- 又大又热的 key 要先拆大，再降热。

下一节我们学习 Pipeline、RTT 和慢命令优化。

