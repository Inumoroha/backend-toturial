# 09. Context、超时、连接池与优雅关闭

这一节是 Go 后端工程味最浓的一节。MongoDB 代码能跑只是第一步，真正可靠的服务必须处理超时、取消、连接池和关闭流程。

## 1. 为什么每个数据库操作都要 context

`context.Context` 可以表达：

- 请求取消。
- 操作超时。
- 上游调用链结束。

MongoDB 操作如果没有超时，网络抖动或数据库阻塞时，请求可能一直占用资源。

推荐方法签名：

```go
func (r *Repository) FindByEmail(ctx context.Context, email string) (*User, error)
```

不要在 Repository 里直接使用：

```go
context.Background()
```

除非是在服务启动、关闭这类没有请求上下文的地方。

## 2. 请求上下文和操作超时

Repository 可以基于传入 ctx 再设置操作超时：

```go
func (r *Repository) FindByEmail(ctx context.Context, email string) (*User, error) {
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()

    // MongoDB query...
}
```

这样既继承上游取消信号，又限制本次数据库操作最长执行时间。

注意：

- 不要所有方法都机械写死 3 秒。
- 查询、写入、批量操作可以使用不同超时。
- 超时策略最好集中配置。

## 3. Client 级 Timeout

Go Driver v2 支持在 Client 上设置操作级 timeout。官方连接选项文档说明：如果操作 Context 没有 deadline，操作会继承 Client 的 Timeout。

示例：

```go
clientOpts := options.Client().
    ApplyURI(uri).
    SetTimeout(5 * time.Second)

client, err := mongo.Connect(clientOpts)
```

如果你每个操作都自己传带 deadline 的 ctx，则以 operation context 为准。

学习阶段建议：

- Client 设置一个兜底 timeout。
- 关键 Repository 方法也显式使用 `context.WithTimeout`。

## 4. 连接池是什么

`mongo.Client` 内部维护连接池。

连接池的作用：

- 复用 TCP 连接。
- 减少每次请求新建连接的成本。
- 控制并发连接数量。

官方连接池文档强调：每个 Client 实例都有内置连接池；为了效率，应每个进程创建一次 Client 并复用。

## 5. 配置连接池

示例：

```go
clientOpts := options.Client().
    ApplyURI(uri).
    SetMaxPoolSize(100).
    SetMinPoolSize(5).
    SetMaxConnIdleTime(5 * time.Minute).
    SetTimeout(5 * time.Second)

client, err := mongo.Connect(clientOpts)
```

常见配置：

| 配置 | 作用 |
| --- | --- |
| `SetMaxPoolSize` | 每个服务器连接池最大连接数 |
| `SetMinPoolSize` | 连接池最小连接数 |
| `SetMaxConnIdleTime` | 空闲连接保留时间 |
| `SetTimeout` | 单次操作兜底超时 |

学习阶段不要乱调太大。默认值通常够用，先理解含义。

## 6. 连接池不是越大越好

连接池太小：

- 并发请求等待连接。
- 延迟升高。

连接池太大：

- MongoDB 连接压力增加。
- 应用占用更多资源。
- 可能让数据库更快被打满。

连接池大小要结合：

- 应用实例数量。
- MongoDB 节点数量。
- 单实例并发。
- 查询耗时。
- 数据库最大连接能力。

## 7. 优雅关闭

真实服务退出时，不应该立刻杀掉进程。应该：

```text
收到退出信号
  -> 停止接收新请求
  -> 等待正在处理的请求结束
  -> 关闭 MongoDB Client
  -> 退出进程
```

学习阶段可以先写简单 defer：

```go
defer func() {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := client.Disconnect(ctx); err != nil {
        log.Printf("disconnect mongodb: %v", err)
    }
}()
```

后面做 Web 项目时再接 `os.Signal` 和 HTTP Server Shutdown。

## 8. 不要滥用 context.Background

错误：

```go
func (r *Repository) Create(ctx context.Context, u *User) error {
    _, err := r.collection.InsertOne(context.Background(), u)
    return err
}
```

这样会丢掉上层请求的取消和超时。

正确：

```go
func (r *Repository) Create(ctx context.Context, u *User) error {
    _, err := r.collection.InsertOne(ctx, u)
    return err
}
```

## 9. 常见超时类型

### 连接超时

连接 MongoDB 过程超时。

### Server Selection Timeout

Driver 找不到可用服务器。

可能原因：

- MongoDB 没启动。
- URI 错误。
- 副本集不可用。
- 网络问题。

### Operation Timeout

单次操作超过限定时间。

可能原因：

- 查询太慢。
- 没有索引。
- 数据量太大。
- 数据库压力高。

## 10. 本节验收

你应该能回答：

- 为什么 Repository 方法要接收 `context.Context`？
- 为什么不要每个请求创建新的 `mongo.Client`？
- `mongo.Client` 和连接池有什么关系？
- `SetMaxPoolSize` 和 `SetMinPoolSize` 分别做什么？
- 什么情况下应该调用 `Disconnect`？
- 为什么不能在 Repository 内随便使用 `context.Background()`？

