# Go Context 后续进阶学习方向

这份文档用于承接当前 `Context` 系统教程。

如果你已经完成了前面 0 到 9 阶段，说明你已经掌握了 Go 后端中 `context` 的核心用法：

```text
取消控制。
超时控制。
请求链路传递。
Context Value。
标准库组件实践。
源码基础理解。
工程规范。
项目实战。
```

接下来可以继续沿着 Go 后端工程能力往下走。

---

## 一、Go 并发模式进阶

`context` 和并发是强绑定的。

建议继续学习：

- goroutine 生命周期管理。
- channel 关闭原则。
- worker pool。
- fan-in / fan-out。
- pipeline 模式。
- select 高级用法。
- `sync.WaitGroup`、`sync.Mutex`、`sync.Once`。
- goroutine 泄漏排查。

重点目标：

```text
能设计一组可启动、可取消、可等待、可安全退出的并发任务。
```

推荐练习：

1. 实现一个支持 `context` 取消的 worker pool。
2. 实现一个 pipeline，任一阶段失败后取消全链路。
3. 使用 `runtime.NumGoroutine()` 和 pprof 观察 goroutine 泄漏。

---

## 二、errgroup 深入使用

前面已经学习过 `errgroup.WithContext`。

后续可以继续深入：

- `errgroup` 和 `WaitGroup` 的区别。
- `SetLimit` 限制并发数。
- 批量任务中的错误处理。
- 并发结果收集。
- 局部失败和整体失败的取舍。
- 可降级任务和关键任务的区分。

推荐练习：

```text
批量查询 100 个用户资料。
最多同时并发 10 个。
任一关键任务失败时取消全部。
非关键任务失败时记录错误但不中断。
```

这个练习非常接近真实后端聚合接口。

---

## 三、HTTP 服务工程化

学完 `context` 后，建议系统学习 Go HTTP 服务。

重点包括：

- `net/http` Server 配置。
- ReadTimeout、WriteTimeout、IdleTimeout。
- Middleware 设计。
- 请求日志。
- Panic recover。
- 参数解析和校验。
- 错误响应格式。
- 优雅关闭。
- Handler、Service、Repository 分层。

需要特别关注：

```text
r.Context() 如何进入业务链路。
HTTP server timeout 和业务 context timeout 的区别。
客户端断开连接时服务端如何处理。
```

推荐练习：

1. 写一个标准库版 REST API。
2. 增加 request id 中间件。
3. 增加超时中间件。
4. 增加统一错误处理。
5. 增加优雅关闭。

---

## 四、数据库与 Context 深入

数据库是 `context` 使用最高频的地方之一。

建议继续学习：

- `database/sql` 的连接池。
- `QueryContext`、`ExecContext`、`BeginTx`。
- PostgreSQL `pgx` 和 `pgxpool`。
- 查询超时和锁等待。
- 事务中的 context。
- rows 关闭和连接归还。
- 慢查询排查。
- 连接池耗尽排查。

推荐练习：

```text
实现一个用户模块：
创建用户。
查询用户。
分页查询。
更新用户。
事务更新用户资料和日志。
所有数据库操作都支持 context timeout。
```

重点不是 CRUD 本身，而是：

```text
所有数据库访问都有明确的生命周期和错误处理。
```

---

## 五、gRPC Deadline 与 Metadata

如果后续学习微服务，gRPC 是很重要的一块。

`context` 在 gRPC 中非常核心：

- deadline 通过 context 传播。
- metadata 通过 context 传递。
- 客户端取消会传到服务端。
- 服务端可以读取 deadline。
- 链路追踪也依赖 context。

建议学习：

- unary RPC 中的 context。
- streaming RPC 中的 context。
- `metadata.NewOutgoingContext`。
- `metadata.FromIncomingContext`。
- gRPC deadline。
- gRPC 拦截器 interceptor。

推荐练习：

1. 写一个 user-service gRPC 服务。
2. client 设置 500ms deadline。
3. server 模拟慢处理。
4. 观察 deadline exceeded。
5. 使用 metadata 传 request id。

---

## 六、OpenTelemetry 链路追踪

`context` 是 OpenTelemetry 在 Go 中传播 trace 的关键载体。

建议学习：

- trace id 和 span id。
- root span 和 child span。
- context 中的 span 传播。
- HTTP middleware 自动注入 trace。
- 数据库和 HTTP client instrumentation。
- 跨服务 trace 传播。

重点理解：

```text
为什么 trace 信息可以跟着 context 从 handler 传到 service、repository、client。
```

推荐练习：

```text
构建一个 API 服务。
接口内部调用一个下游 HTTP 服务。
接入 OpenTelemetry。
在 Jaeger 或其他后端中看到完整调用链。
```

---

## 七、服务治理：超时、重试、熔断、限流

`context` 解决的是生命周期边界，但真实服务还需要更多治理能力。

建议继续学习：

- 超时预算。
- 重试策略。
- 指数退避。
- 熔断。
- 限流。
- 降级。
- 舱壁隔离。

需要注意：

```text
重试必须尊重 context 总预算。
熔断和限流不能破坏请求取消语义。
降级逻辑要区分关键下游和非关键下游。
```

推荐练习：

```text
实现一个聚合接口。
订单服务失败时降级为空列表。
钱包服务失败时返回错误。
所有下游调用共享总 timeout。
重试最多发生在总预算内。
```

---

## 八、日志、监控与排障

学会 `context` 后，要继续学习如何观察它。

建议学习：

- 结构化日志。
- request id。
- trace id。
- 错误分类。
- latency histogram。
- timeout 指标。
- goroutine 数量监控。
- 连接池监控。
- pprof。

推荐练习：

1. 给 HTTP 服务接入结构化日志。
2. 每条日志带 request id。
3. 记录下游调用耗时。
4. 统计 timeout 次数。
5. 使用 pprof 查看 goroutine。

目标是：

```text
线上出现 context deadline exceeded 时，你能知道是哪个接口、哪个下游、耗时多少、是否连接池耗尽。
```

---

## 九、Context 源码深入

当前教程只做了源码入门。

后续可以继续深入：

- Go 当前版本 `context.go` 完整阅读。
- `cancelCtx` 的并发安全。
- `Done` channel 的延迟创建。
- `propagateCancel` 的不同分支。
- `AfterFunc`。
- `WithCancelCause`。
- `Cause`。
- `WithoutCancel`。

这些 API 在新版本 Go 中很有用。

建议专题学习：

```text
WithCancelCause 和普通 WithCancel 的区别。
WithoutCancel 适合什么场景。
AfterFunc 如何在 context 取消时执行回调。
```

---

## 十、项目进阶方向

完成当前三个 Context 项目后，可以继续做更完整的项目。

### 项目一：带 Context 的用户中心

功能：

- 用户注册。
- 用户登录。
- 查询当前用户。
- 更新资料。
- PostgreSQL 存储。
- request id 日志。
- 数据库查询 timeout。
- 优雅关闭。

### 项目二：订单聚合 API

功能：

- 查询订单详情。
- 并发查询商品、库存、支付、物流。
- 使用 errgroup。
- 非关键下游可降级。
- 关键下游失败返回错误。
- 统一 timeout budget。

### 项目三：后台任务系统

功能：

- worker pool。
- job queue。
- context 控制任务取消。
- signal 优雅关闭。
- 任务超时。
- 失败重试。
- 退出时等待任务完成。

### 项目四：微服务 Trace Demo

功能：

- API Gateway。
- User Service。
- Order Service。
- gRPC 或 HTTP 调用。
- request id。
- OpenTelemetry trace。
- context deadline 传播。

---

## 十一、推荐学习顺序

如果你想继续系统推进，可以按这个顺序：

```text
Go 并发模式
  -> errgroup 深入
  -> HTTP 服务工程化
  -> 数据库与连接池
  -> gRPC deadline 和 metadata
  -> OpenTelemetry
  -> 服务治理
  -> 日志监控排障
  -> 综合项目
```

不要同时学太多。

建议每学一个方向，都做一个小项目。

---

## 十二、最终能力目标

继续进阶后，你应该逐步达到：

- 能设计清晰的请求生命周期。
- 能为接口和下游调用设计 timeout budget。
- 能写不会轻易泄漏 goroutine 的并发代码。
- 能优雅关闭 HTTP 服务和 worker。
- 能在数据库、HTTP、gRPC 中正确传递 context。
- 能用日志、指标、trace 排查超时和取消问题。
- 能把 context 和服务治理能力结合起来。

这时你对 `context` 的掌握就不只是 API 层面，而是 Go 后端工程层面的能力。
