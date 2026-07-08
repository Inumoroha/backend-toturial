# 5. MEMORY USAGE 与 MEMORY STATS：内存异常排查

Redis 是内存系统。

内存异常是生产排查中的高频问题：内存上涨、淘汰增加、碎片率高、Big Key、key 无 TTL、AOF/RDB COW 压力等。

学完这一节后，你应该能够：

- 使用 `INFO memory` 查看整体内存。
- 使用 `MEMORY USAGE` 查看单 key 内存。
- 使用 `MEMORY STATS` 查看内存细节。
- 理解内存碎片。
- 建立内存异常排查路径。

---

## 一、先看 INFO memory

```redis
INFO memory
```

关注：

```text
used_memory_human
used_memory_peak_human
maxmemory_human
mem_fragmentation_ratio
allocator_frag_ratio
```

判断：

- 当前内存是否接近上限。
- 峰值是否异常。
- 碎片率是否过高。
- 是否发生淘汰。

---

## 二、MEMORY USAGE

查看单个 key 占用：

```redis
MEMORY USAGE article:detail:1001
```

返回字节数。

示例：

```text
2048
```

表示约 2KB。

它是估算值，但足够用于排查 Big Key。

---

## 三、集合类型还要看元素数量

不同类型：

```redis
STRLEN string:key
HLEN hash:key
LLEN list:key
SCARD set:key
ZCARD zset:key
```

元素数量和内存都要看。

例如：

```text
ZCARD = 1000000
MEMORY USAGE = 120MB
```

这就是明显 Big Key。

---

## 四、MEMORY STATS

```redis
MEMORY STATS
```

可以查看更细的内存统计。

关注方向：

- 数据占用。
- allocator 碎片。
- RSS。
- key 元数据。

不同 Redis 版本输出字段可能略有差异。

不要死背字段，重点是理解内存是否被数据、碎片或额外开销占用。

---

## 五、内存碎片是什么

简单理解：

```text
Redis 申请和释放内存后，操作系统看到的占用可能大于 Redis 实际数据占用。
```

指标：

```text
mem_fragmentation_ratio
```

如果明显大于 1，说明有碎片或 RSS 额外占用。

轻微大于 1 正常。

如果长期很高，要进一步分析。

---

## 六、碎片率高怎么办

可能原因：

- key 频繁创建删除。
- value 大小变化频繁。
- 大 key 删除。
- 内存分配器碎片。

处理：

- 观察趋势。
- 避免大 key 频繁变更。
- 使用 `UNLINK` 删除大 key。
- 开启或调整 active defrag。
- 必要时主从切换或重启释放碎片。

重启是最后手段，要有高可用和演练。

---

## 七、查看 key 是否有 TTL

```redis
TTL key
```

如果缓存 key 返回：

```text
-1
```

说明它没有过期时间。

大量缓存 key 没 TTL，会导致内存持续上涨。

可以通过抽样扫描检查：

```redis
SCAN 0 MATCH article:detail:* COUNT 100
```

然后对样本执行 `TTL`。

---

## 八、内存上涨排查路径

```text
1. INFO memory 看 used_memory 趋势。
2. INFO stats 看 evicted_keys 是否增长。
3. INFO keyspace 看 key 数量和 expires 数量。
4. 抽样 SCAN 看 key 命名和 TTL。
5. MEMORY USAGE 找大 key。
6. 查看最近上线或批任务。
7. 判断是正常增长、泄漏式增长还是热点写入。
8. 制定清理、拆分或 TTL 策略。
```

不要直接全库 `KEYS *`。

---

## 九、内存满和淘汰

如果内存接近 `maxmemory`：

```redis
INFO stats
```

看：

```text
evicted_keys
```

如果持续增长，说明 Redis 正在淘汰。

缓存场景可能还能接受。

如果 Redis 保存重要状态，被淘汰就是数据丢失。

---

## 十、COW 内存压力

RDB、AOF 重写、全量复制可能 fork 子进程。

写入较多时 COW 会增加内存。

查看：

```redis
INFO persistence
```

关注：

```text
rdb_last_cow_size
aof_last_cow_size
```

如果故障发生在 `BGSAVE` 或 `BGREWRITEAOF` 附近，要考虑 COW 内存压力。

---

## 十一、Go 中避免内存问题

常见建议：

- 不缓存巨大 JSON。
- 列表缓存只存 ID。
- 给缓存设置 TTL。
- 对大集合分页读取。
- 避免一次写入超大 value。
- 对批量结果设置上限。

示例：

```text
home:feed:ids -> ID 列表
article:detail:{id} -> 单篇详情
```

比一个巨大 `home:feed` JSON 更可控。

---

## 十二、常见错误

### 1. 只看 key 数量，不看 key 大小

少量 Big Key 也能占很多内存。

### 2. 缓存 key 没 TTL

内存长期增长。

### 3. 使用 KEYS 排查

生产应使用 SCAN 抽样。

### 4. 碎片率高就立刻重启

先分析趋势和业务影响。

### 5. 忽略 COW

持久化或复制期间内存可能临时上升。

---

## 十三、本节练习

请完成下面练习：

1. 查看 `INFO memory`。
2. 查看一个 key 的 `MEMORY USAGE`。
3. 对 String 使用 `STRLEN`。
4. 对 Hash 使用 `HLEN`。
5. 查看 `MEMORY STATS`。
6. 抽样检查一批缓存 key 的 TTL。
7. 设计一个内存上涨排查流程。
8. 思考你项目中哪些 key 可能没有 TTL。

---

## 十四、本节小结

这一节你学习了 Redis 内存排查。

你需要记住：

- `INFO memory` 看整体内存。
- `MEMORY USAGE` 看单 key 内存。
- 集合类型还要看元素数量。
- 内存上涨要结合 key 数量、TTL、Big Key、淘汰和 COW 分析。
- 缓存 key 应有合理生命周期。

下一节我们学习 Big Key 和 Hot Key 的发现与治理。

