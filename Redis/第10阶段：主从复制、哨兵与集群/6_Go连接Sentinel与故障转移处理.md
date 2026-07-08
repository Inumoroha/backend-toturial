# 6. Go 连接 Sentinel 与故障转移处理

使用 Sentinel 后，应用不能再固定连接某一个 Redis master 地址。

客户端需要通过 Sentinel 发现当前 master，并在故障转移后自动切换。

学完这一节后，你应该能够：

- 使用 go-redis 的 `NewFailoverClient`。
- 理解 master name 的作用。
- 配置 Sentinel 地址列表。
- 知道故障转移期间客户端会遇到什么。
- 为 Redis 操作设置超时、重试和降级。

---

## 一、为什么不能固定连 master

主从 + Sentinel 架构下：

```text
故障前 master = redis-1
故障后 master = redis-2
```

如果应用配置写死：

```text
redis-1:6379
```

故障转移后应用还会连接旧地址。

正确做法是：

```text
应用连接 Sentinel，通过 master name 获取当前 master。
```

---

## 二、go-redis FailoverClient

示例：

```go
rdb := redis.NewFailoverClient(&redis.FailoverOptions{
    MasterName:    "mymaster",
    SentinelAddrs: []string{"sentinel-1:26379", "sentinel-2:26379", "sentinel-3:26379"},
    DB:            0,
})
```

`MasterName` 要和 Sentinel 配置一致：

```conf
sentinel monitor mymaster redis-master 6379 2
```

客户端会通过 Sentinel 查询当前 master。

---

## 三、带密码的配置

如果 Redis master/replica 有密码：

```go
rdb := redis.NewFailoverClient(&redis.FailoverOptions{
    MasterName:       "mymaster",
    SentinelAddrs:    []string{"sentinel-1:26379", "sentinel-2:26379", "sentinel-3:26379"},
    Password:         "redis-password",
    SentinelPassword: "sentinel-password",
})
```

Redis 密码和 Sentinel 密码可能不同。

要分清楚。

---

## 四、健康检查

启动时可以：

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

if err := rdb.Ping(ctx).Err(); err != nil {
    return err
}
```

但不要只依赖启动时检查。

运行中 Redis 仍可能切换或抖动。

每次操作都要有 context 超时。

---

## 五、操作超时配置

```go
rdb := redis.NewFailoverClient(&redis.FailoverOptions{
    MasterName:    "mymaster",
    SentinelAddrs: []string{"sentinel-1:26379", "sentinel-2:26379", "sentinel-3:26379"},

    DialTimeout:  500 * time.Millisecond,
    ReadTimeout:  200 * time.Millisecond,
    WriteTimeout: 200 * time.Millisecond,
    PoolSize:     50,
    MinIdleConns: 10,
})
```

示例值要根据实际环境调整。

原则：

```text
Redis 操作不能无限拖住业务请求。
```

---

## 六、故障转移期间会发生什么

故障转移期间，应用可能遇到：

- 连接断开。
- 写入失败。
- 短暂找不到 master。
- READONLY 错误。
- 请求超时。
- 重试后成功。

所以业务代码要把 Redis 当成可能失败的外部依赖。

缓存场景可以降级。

锁、限流、任务队列等场景要更谨慎。

---

## 七、缓存场景的降级

读取缓存失败：

```go
val, err := rdb.Get(ctx, key).Result()
if errors.Is(err, redis.Nil) {
    return loadFromDB(ctx)
}
if err != nil {
    logger.Printf("redis get failed key=%s err=%v", key, err)
    return loadFromDB(ctx)
}
```

写缓存失败：

```go
if err := rdb.Set(ctx, key, val, ttl).Err(); err != nil {
    logger.Printf("redis set failed key=%s err=%v", key, err)
}
```

缓存失败通常不应该让主业务直接失败。

但要防止大量回源压垮数据库。

---

## 八、非缓存场景要谨慎

如果 Redis 用于：

- 分布式锁。
- 限流。
- 幂等。
- Stream 任务队列。
- 库存预扣。

故障转移期间的错误不能简单忽略。

例如限流 Redis 失败：

- 登录接口可能要保守拒绝或增加验证码。
- 短信接口可能要拒绝发送。
- 普通阅读计数可以丢弃。

要按业务风险制定策略。

---

## 九、重试要配合幂等

故障转移期间重试是常见策略。

但重试会带来重复执行问题。

例如：

```text
请求超时，但 Redis 实际写入成功。
客户端重试，又执行一次。
```

对于 `INCR`、发消息、扣库存等操作，要考虑幂等或可重复语义。

不要盲目无限重试。

---

## 十、监控客户端错误

建议记录：

- Redis 超时次数。
- READONLY 错误次数。
- 连接重建次数。
- Sentinel 查询失败次数。
- 命令重试次数。
- 降级次数。

故障转移不是只看 Redis 服务端。

客户端指标也很关键。

---

## 十一、常见错误

### 1. Sentinel 架构下仍连接固定 master

故障转移后应用不会自动切换。

### 2. MasterName 配错

客户端无法从 Sentinel 找到 master。

### 3. 不配置超时

Redis 抖动时拖住业务请求。

### 4. 所有 Redis 错误都吞掉

限流、锁、任务场景会出业务风险。

### 5. 无限重试

可能放大故障和重复执行。

---

## 十二、本节练习

请完成下面练习：

1. 使用 `redis.NewFailoverClient` 创建客户端。
2. 配置 3 个 Sentinel 地址。
3. 设置 `MasterName`。
4. 为 Redis 操作设置 context 超时。
5. 手动停止 master，观察客户端是否恢复。
6. 记录故障转移期间出现的错误。
7. 为缓存、限流、锁分别设计降级策略。

---

## 十三、本节小结

这一节你学习了 Go 如何连接 Sentinel。

你需要记住：

- Sentinel 架构下客户端要通过 master name 发现当前 master。
- go-redis 使用 `NewFailoverClient`。
- 故障转移期间会出现短暂错误和连接重建。
- Redis 操作必须有超时。
- 缓存、限流、锁、任务队列的错误处理策略不同。

下一节我们学习 Redis Cluster、Hash Slot、MOVED 和 ASK。

