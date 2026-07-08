# 2. redis.Client 连接池配置：PoolSize 与 MinIdleConns

`go-redis/v9` 的 `redis.Client` 内部带连接池。

这意味着你不需要自己手动管理每条 Redis TCP 连接，但你需要理解连接池配置，否则高并发场景下可能出现连接池耗尽、请求排队、超时等问题。

学完这一节后，你应该能够：

- 理解为什么 Redis client 需要连接池。
- 配置 `PoolSize` 和 `MinIdleConns`。
- 理解连接池太小或太大分别有什么问题。
- 知道 Redis 操作超时和连接池等待的关系。
- 为本地学习和普通 Go 服务设置合理初始值。

---

## 一、为什么需要连接池

Go 服务通常会同时处理多个 HTTP 请求。

每个请求可能都要访问 Redis。

如果每次 Redis 操作都新建 TCP 连接：

```text
建立连接 -> 执行命令 -> 关闭连接
```

成本会很高。

连接池的作用是：

```text
提前维护一批连接
请求需要 Redis 时从池里借连接
用完归还连接
```

`go-redis` 的 `redis.Client` 已经帮你做了这件事。

所以正确姿势是：创建一个长期复用的 `redis.Client`，让它内部管理连接池。

---

## 二、最小连接池配置

示例：

```go
rdb := redis.NewClient(&redis.Options{
    Addr:         "127.0.0.1:6379",
    Password:     "",
    DB:           0,
    PoolSize:     20,
    MinIdleConns: 5,
})
```

这里重点是：

| 字段 | 含义 |
| --- | --- |
| `PoolSize` | 连接池最大连接数 |
| `MinIdleConns` | 最小空闲连接数 |

`PoolSize` 控制最多可以同时有多少连接用于 Redis 操作。

`MinIdleConns` 控制池里尽量保留多少空闲连接，减少突发请求时临时建连接的成本。

---

## 三、PoolSize 怎么理解

`PoolSize` 不是 Redis 最大 QPS，也不是 HTTP 并发数。

它表示这个 client 连接池最多能持有多少 Redis 连接。

假设：

```go
PoolSize: 20
```

那么同一时刻最多大约有 20 个 Redis 连接可以执行命令。

如果并发请求很多，超过连接池能力的请求可能会等待可用连接。

等待太久，就可能超时。

---

## 四、PoolSize 太小的问题

如果 `PoolSize` 太小，可能出现：

- Redis 操作排队等待连接。
- HTTP 接口延迟升高。
- 请求超时。
- 日志里出现连接池超时相关错误。

例如服务每秒请求很多，每个请求要访问 Redis 多次，但连接池只有 2 个连接，就容易排队。

本地学习时 `PoolSize: 10` 通常够了。

普通小服务可以从 `20` 或 `50` 开始，根据监控调整。

---

## 五、PoolSize 太大的问题

连接池不是越大越好。

如果 `PoolSize` 太大，可能导致：

- Redis 服务端连接数过多。
- Go 服务占用更多资源。
- 故障时更多请求同时打 Redis。
- 多个服务实例叠加后连接数失控。

比如你有 20 个 Go 服务实例，每个实例 `PoolSize: 200`，理论上可能产生 4000 个 Redis 连接。

Redis 本身、网络、操作系统都要承受这些连接。

所以连接池大小要结合服务实例数一起看。

---

## 六、MinIdleConns 怎么理解

`MinIdleConns` 表示连接池中尽量保留的空闲连接数。

示例：

```go
MinIdleConns: 5
```

它的作用是：当请求突然到来时，池里已经有一些可用连接，减少临时建连接的延迟。

如果服务流量比较稳定，可以设置一个小的 `MinIdleConns`。

如果流量很低，本地学习甚至可以不配置。

---

## 七、推荐初始配置

本地学习：

```go
rdb := redis.NewClient(&redis.Options{
    Addr:         "127.0.0.1:6379",
    PoolSize:     10,
    MinIdleConns: 2,
})
```

普通 Web 服务起步：

```go
rdb := redis.NewClient(&redis.Options{
    Addr:         "redis:6379",
    PoolSize:     20,
    MinIdleConns: 5,
})
```

高并发服务：

```text
不要只靠猜。
根据 QPS、Redis 命令耗时、服务实例数、Redis 连接数监控来调整。
```

连接池配置没有万能值。

---

## 八、超时配置也很重要

除了连接池，还应该配置超时。

示例：

```go
rdb := redis.NewClient(&redis.Options{
    Addr:         "127.0.0.1:6379",
    PoolSize:     20,
    MinIdleConns: 5,
    DialTimeout:  2 * time.Second,
    ReadTimeout:  1 * time.Second,
    WriteTimeout: 1 * time.Second,
    PoolTimeout:  2 * time.Second,
})
```

常见字段：

| 字段 | 含义 |
| --- | --- |
| `DialTimeout` | 建立连接超时 |
| `ReadTimeout` | 读取 Redis 响应超时 |
| `WriteTimeout` | 写入 Redis 命令超时 |
| `PoolTimeout` | 等待连接池可用连接的超时 |

这些配置能避免请求在 Redis 异常时无限等待。

---

## 九、PoolTimeout 是什么

当连接池里没有可用连接时，请求会等待。

`PoolTimeout` 控制最多等多久。

如果超过时间还拿不到连接，就返回错误。

这通常说明：

- Redis 命令太慢。
- 连接池太小。
- 并发太高。
- 某些请求没有及时释放连接。
- Redis 或网络异常。

遇到连接池超时，不要第一反应就把 `PoolSize` 调很大。

应该先看：

- Redis 命令耗时。
- 慢日志。
- 接口并发。
- 客户端超时配置。
- 是否有大 value 或慢命令。

---

## 十、连接池配置与 context 的关系

连接池配置是 client 层面的默认限制。

`context.Context` 是每次请求的取消和超时控制。

一个 Redis 操作可能同时受两类限制：

```text
client ReadTimeout / PoolTimeout
request context timeout
```

比如 HTTP 请求总超时是 200ms，那么 Redis 操作也不应该等 1 秒。

下一节会专门讲 context。

这里先记住：连接池配置和 context 是配合关系，不是二选一。

---

## 十一、如何观察连接池状态

`go-redis` 可以获取连接池统计：

```go
stats := rdb.PoolStats()
fmt.Printf("hits=%d misses=%d timeouts=%d total=%d idle=%d stale=%d\n",
    stats.Hits,
    stats.Misses,
    stats.Timeouts,
    stats.TotalConns,
    stats.IdleConns,
    stats.StaleConns,
)
```

这些指标可以帮助你判断：

- 连接池是否经常超时。
- 空闲连接是否太多或太少。
- 总连接数是否接近上限。

生产环境应该把这些指标接入监控。

---

## 十二、常见错误

### 1. 每个请求创建 client

这样每个请求都有自己的连接池，资源浪费严重。

### 2. PoolSize 随便调很大

多个服务实例叠加后，Redis 连接数可能过高。

### 3. 没有超时配置

Redis 异常时，请求可能等待过久。

### 4. 忽略 PoolTimeout

连接池等待超时通常说明系统已经有压力，需要排查。

### 5. 只调客户端，不看 Redis 服务端

Redis 慢命令、Big Key、网络抖动都可能导致连接池问题。

---

## 十三、本节练习

请完成下面练习：

1. 在 `redis.NewClient` 中配置 `PoolSize: 10`。
2. 配置 `MinIdleConns: 2`。
3. 配置 `DialTimeout`、`ReadTimeout`、`WriteTimeout`、`PoolTimeout`。
4. 使用 `Ping` 验证连接。
5. 打印 `rdb.PoolStats()`。
6. 思考为什么不能每个请求创建 client。
7. 思考 `PoolSize` 太小会发生什么。
8. 思考 `PoolSize` 太大会发生什么。
9. 如果有 10 个服务实例，每个实例 `PoolSize` 是 100，最多可能有多少 Redis 连接？
10. 连接池超时时，你会先排查哪些问题？

---

## 十四、本节小结

这一节你学习了 `redis.Client` 的连接池配置。

你需要记住：

- `redis.Client` 内部管理连接池，应该长期复用。
- `PoolSize` 控制最大连接数，不是越大越好。
- `MinIdleConns` 可以保留一定空闲连接，减少突发建连成本。
- `DialTimeout`、`ReadTimeout`、`WriteTimeout`、`PoolTimeout` 能保护请求不被 Redis 长时间拖住。
- 连接池配置要结合服务实例数、Redis 服务端能力和实际监控调整。
- 可以用 `PoolStats()` 观察连接池状态。

下一节我们学习 `context.Context`，它是 Go 服务中控制 Redis 操作超时和取消信号的关键。
