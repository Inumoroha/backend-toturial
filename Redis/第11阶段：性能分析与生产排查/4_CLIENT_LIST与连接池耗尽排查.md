# 4. CLIENT LIST 与连接池耗尽排查

Redis 性能问题不一定来自 Redis 命令慢。

很多接口超时来自客户端连接池耗尽、连接过多、阻塞连接或网络异常。

学完这一节后，你应该能够：

- 使用 `CLIENT LIST` 查看连接。
- 理解连接数异常的常见原因。
- 排查 Go 连接池耗尽。
- 配置 go-redis 连接池参数。
- 知道连接问题如何影响业务延迟。

---

## 一、查看连接数

```redis
INFO clients
```

关注：

```text
connected_clients
blocked_clients
```

如果 `connected_clients` 突然升高，说明连接数量异常。

如果 `blocked_clients` 升高，可能有阻塞命令。

---

## 二、CLIENT LIST

```redis
CLIENT LIST
```

会列出当前客户端连接。

每条连接包含：

- 地址。
- 连接持续时间。
- 空闲时间。
- 当前 db。
- 最后执行命令。
- 输入输出缓冲区。
- 客户端名称。

输出较多时要谨慎。

连接很多的生产实例上不要频繁执行。

---

## 三、给客户端命名

Go 客户端可以设置：

```go
rdb := redis.NewClient(&redis.Options{
    Addr:       "127.0.0.1:6379",
    ClientName: "article-service",
})
```

这样 `CLIENT LIST` 里能看到服务名。

排查多服务共用 Redis 时非常有用。

---

## 四、连接数异常原因

常见原因：

- 应用实例数增加。
- 每个实例连接池太大。
- 连接泄漏。
- 短连接频繁创建。
- Redis 超时导致重连风暴。
- Sentinel/Cluster 切换后连接重建。
- 阻塞命令占住连接。

连接数不是越大越好。

过多连接会增加 Redis 和操作系统负担。

---

## 五、连接池耗尽是什么

Go 服务通常使用连接池。

如果请求需要 Redis 连接，但池里没有可用连接，就要等待。

表现为：

```text
应用侧 Redis 操作耗时很高，
但 Redis SLOWLOG 没有慢命令。
```

原因是慢在等连接，不是慢在 Redis 执行。

---

## 六、go-redis 连接池配置

```go
rdb := redis.NewClient(&redis.Options{
    Addr:         "127.0.0.1:6379",
    PoolSize:     50,
    MinIdleConns: 10,
    PoolTimeout:  100 * time.Millisecond,
    DialTimeout:  500 * time.Millisecond,
    ReadTimeout:  200 * time.Millisecond,
    WriteTimeout: 200 * time.Millisecond,
})
```

关键参数：

- `PoolSize`：最大连接数。
- `MinIdleConns`：最小空闲连接。
- `PoolTimeout`：等待连接最长时间。
- `DialTimeout`：建连接超时。
- `ReadTimeout` / `WriteTimeout`：读写超时。

---

## 七、如何设置 PoolSize

不能只拍脑袋。

要结合：

- 应用实例数。
- 每个实例并发。
- Redis 操作耗时。
- Redis 最大连接数。
- Redis QPS。

例如：

```text
20 个应用实例
每个 PoolSize=100
总连接上限约 2000
```

如果 Redis `maxclients` 是 10000，看似够。

但连接过多仍会增加调度和内存开销。

---

## 八、查看连接池状态

go-redis 提供：

```go
stats := rdb.PoolStats()
```

常见字段：

```go
stats.Hits
stats.Misses
stats.Timeouts
stats.TotalConns
stats.IdleConns
stats.StaleConns
```

关注：

- `Timeouts` 是否增长。
- `TotalConns` 是否接近 `PoolSize`。
- `IdleConns` 是否长期为 0。

这些指标应该接入监控。

---

## 九、阻塞命令占连接

例如：

```redis
BLPOP queue 0
XREAD BLOCK 0 STREAMS stream $
```

阻塞命令会长期占用连接。

不要让普通命令和阻塞命令共用同一个小连接池。

可以为 Stream worker、阻塞队列单独创建 Redis client。

---

## 十、连接池和 Pipeline

没有 Pipeline 时：

```text
100 个 Redis 命令 = 100 次网络往返
```

这些命令会占用连接更久。

Pipeline 可以减少 RTT，也能降低连接占用时间。

但 Pipeline 不要一次塞太大，避免大响应和阻塞。

---

## 十一、排查路径

应用 Redis 操作超时：

```text
1. 看 SLOWLOG 有没有慢命令。
2. 如果没有，看应用侧连接池 stats。
3. 看 PoolTimeout 是否增长。
4. 看 CLIENT LIST 连接来源。
5. 看是否有阻塞命令。
6. 看网络 RTT。
7. 优化 PoolSize、超时、Pipeline 或拆分 client。
```

这条路径能避免误判。

---

## 十二、常见错误

### 1. 每次请求创建 Redis client

会导致大量短连接和资源浪费。

### 2. PoolSize 越大越好

过大连接数会增加 Redis 压力。

### 3. 不监控 PoolStats

连接池耗尽时只看到接口超时。

### 4. 阻塞命令和普通命令共用连接池

阻塞命令会占住连接。

### 5. 没有设置 PoolTimeout

请求可能长时间等待连接。

---

## 十三、本节练习

请完成下面练习：

1. 执行 `INFO clients`。
2. 执行 `CLIENT LIST`。
3. 给 go-redis 设置 `ClientName`。
4. 打印 `rdb.PoolStats()`。
5. 观察 `Timeouts` 是否增长。
6. 为阻塞消费单独创建 Redis client。
7. 解释为什么 SLOWLOG 没有慢命令但应用仍可能超时。

---

## 十四、本节小结

这一节你学习了连接和连接池排查。

你需要记住：

- `CLIENT LIST` 可以查看连接来源和状态。
- 应用超时可能是等待连接，不是 Redis 命令慢。
- go-redis 的 `PoolStats` 很重要。
- 阻塞命令应该使用单独 client。
- PoolSize、PoolTimeout、读写超时都要结合业务配置。

下一节我们学习内存排查：`MEMORY USAGE` 和 `MEMORY STATS`。

