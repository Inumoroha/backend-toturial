# 6. 内存淘汰策略：noeviction、allkeys-lru、volatile-lfu

Redis 是内存数据库。

当内存达到上限时，Redis 必须决定：拒绝写入，还是删除一些 key 腾空间。

这个决定由 `maxmemory-policy` 控制。

学完这一节后，你应该能够：

- 配置 `maxmemory`。
- 理解常见淘汰策略。
- 区分 `allkeys-*` 和 `volatile-*`。
- 判断缓存场景适合什么策略。
- 理解写入报错和 key 被淘汰的业务影响。

---

## 一、maxmemory 是什么

配置：

```conf
maxmemory 1gb
```

表示 Redis 最多使用约 1GB 内存。

也可以运行时设置：

```redis
CONFIG SET maxmemory 1073741824
```

如果不配置，Redis 可能持续使用内存，直到机器内存紧张。

生产中通常应该明确配置。

---

## 二、maxmemory-policy 是什么

配置：

```conf
maxmemory-policy allkeys-lru
```

当内存达到 `maxmemory` 时，Redis 根据这个策略处理。

常见行为：

- 淘汰一些 key。
- 或者拒绝写命令。

查询：

```redis
CONFIG GET maxmemory-policy
```

---

## 三、noeviction

```conf
maxmemory-policy noeviction
```

含义：

```text
不淘汰 key，内存满后写命令报错。
```

读命令通常还能执行。

适合：

- 不希望 Redis 自动删除任何 key。
- Redis 保存比较重要的状态。
- 希望业务显式处理写入失败。

不适合纯缓存场景。

缓存内存满了直接写失败，通常会影响请求链路。

---

## 四、allkeys-lru

```conf
maxmemory-policy allkeys-lru
```

含义：

```text
从所有 key 中选择最近最少使用的 key 淘汰。
```

适合：

- Redis 主要作为缓存。
- key 不一定都有 TTL。
- 希望保留最近常用数据。

这是非常常见的缓存策略。

但要记住：Redis 的 LRU 是近似 LRU，不是严格全局排序。

---

## 五、volatile-lru

```conf
maxmemory-policy volatile-lru
```

含义：

```text
只从设置了过期时间的 key 中按 LRU 淘汰。
```

如果很多 key 没有 TTL，它们不会被淘汰。

风险：

```text
如果可淘汰的 TTL key 不够，写入仍可能失败。
```

适合：

- 你明确区分了可淘汰缓存 key 和不可淘汰 key。
- 所有缓存 key 都设置了 TTL。

---

## 六、allkeys-lfu

```conf
maxmemory-policy allkeys-lfu
```

含义：

```text
从所有 key 中选择使用频率较低的 key 淘汰。
```

LFU 更关注访问频率。

适合：

- 有明显热点数据。
- 希望保留高频访问 key。
- 访问模式不是简单的最近访问。

例如热门商品、热门文章缓存。

---

## 七、volatile-lfu

```conf
maxmemory-policy volatile-lfu
```

含义：

```text
只从设置 TTL 的 key 中按 LFU 淘汰。
```

和 `volatile-lru` 一样，它只处理带过期时间的 key。

如果 Redis 中很多 key 没 TTL，内存满时选择空间会很小。

---

## 八、random 和 ttl 策略

随机淘汰：

```conf
allkeys-random
volatile-random
```

TTL 淘汰：

```conf
volatile-ttl
```

`volatile-ttl` 会优先淘汰剩余 TTL 较短的 key。

这些策略相对少见。

缓存业务通常先考虑：

```text
allkeys-lru
allkeys-lfu
```

---

## 九、allkeys 和 volatile 怎么选

简单判断：

```text
Redis 只做缓存 -> allkeys-lru 或 allkeys-lfu
Redis 混合保存缓存和重要状态 -> 谨慎，最好拆实例
只允许淘汰带 TTL 的缓存 -> volatile-lru 或 volatile-lfu
```

更推荐的生产思路：

```text
不同用途的 Redis 分开实例或分开库。
不要把重要状态和可淘汰缓存混在一起。
```

否则淘汰策略会很难选。

---

## 十、淘汰后业务会发生什么

被淘汰的 key 对业务表现为：

```text
缓存未命中。
```

所以业务代码必须能处理：

```text
GET 返回 nil
```

缓存回源数据库后重新写入 Redis。

如果你把 Redis 当作唯一存储，一旦 key 被淘汰，数据就没了。

这就是为什么重要数据不要放在会淘汰的 Redis 中。

---

## 十一、观察淘汰情况

```redis
INFO stats
```

关注：

```text
evicted_keys
```

如果 `evicted_keys` 持续增长，说明 Redis 正在因为内存压力淘汰 key。

还可以查看：

```redis
INFO memory
```

关注：

```text
used_memory_human
maxmemory_human
mem_fragmentation_ratio
```

---

## 十二、Go 中处理缓存被淘汰

读取缓存：

```go
val, err := rdb.Get(ctx, key).Result()
if errors.Is(err, redis.Nil) {
    // 缓存未命中，可能是过期、淘汰或从未写入
    return loadFromDB(ctx)
}
if err != nil {
    return fallback(ctx, err)
}
```

代码不需要区分：

```text
过期未命中
淘汰未命中
从未存在
```

多数缓存场景统一回源即可。

---

## 十三、常见错误

### 1. 不设置 maxmemory

Redis 可能占满机器内存。

### 2. 缓存场景使用 noeviction

内存满后写缓存会报错。

### 3. 使用 volatile 策略但很多 key 没 TTL

可淘汰 key 不够，写入仍可能失败。

### 4. 重要数据和缓存混用同一实例

淘汰策略很难同时满足两种业务。

### 5. 不监控 evicted_keys

Redis 已经开始淘汰，但业务方毫无感知。

---

## 十四、本节练习

请完成下面练习：

1. 查看当前 `maxmemory`。
2. 查看当前 `maxmemory-policy`。
3. 设置一个较小的 `maxmemory`。
4. 分别测试 `noeviction` 和 `allkeys-lru` 的行为。
5. 查看 `INFO stats` 中的 `evicted_keys`。
6. 解释 `allkeys-lru` 和 `volatile-lru` 的区别。
7. 思考缓存和重要状态是否应该放在同一个 Redis 实例。

---

## 十五、本节小结

这一节你学习了内存淘汰策略。

你需要记住：

- `maxmemory` 控制 Redis 内存上限。
- `maxmemory-policy` 控制内存满时如何处理。
- `noeviction` 会拒绝写入，不自动淘汰。
- `allkeys-lru/lfu` 适合纯缓存场景。
- `volatile-lru/lfu` 只淘汰设置 TTL 的 key。
- 重要数据不要放在会被淘汰的 Redis 中。

下一节我们继续深入 LRU、LFU 的近似实现，并讨论 Big Key。

