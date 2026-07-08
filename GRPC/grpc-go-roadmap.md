# Go 后端工程师 gRPC 系统学习路线图

> 目标：从能跑通 gRPC demo，逐步学习到能在真实 Go 后端项目里设计、开发、测试和运维 gRPC 服务。
>
> 建议周期：8-10 周。每天 1-2 小时可以完成第一轮；如果要做综合项目，建议留 12 周。

## 1. 学习前置条件

在系统学习 gRPC 前，建议先掌握这些 Go 后端基础：

- `context.Context`：超时、取消、请求链路传递。
- goroutine 和 channel：理解并发、阻塞和流式处理。
- interface：理解生成代码中的 server/client 接口。
- error wrapping：能区分业务错误和系统错误。
- Go module：会使用 `go mod init`、`go get`、`go mod tidy`。
- HTTP 基础：请求响应、状态码、Header、TLS。

必备工具：

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

Windows PowerShell 临时加入 PATH：

```powershell
$env:Path += ";$(go env GOPATH)\bin"
```

常用依赖：

```bash
go get google.golang.org/grpc
go get google.golang.org/protobuf
```

## 2. 总体学习路径

```text
Go 后端基础
  -> Protocol Buffers
  -> gRPC 基础 RPC
  -> 四种 RPC 模式
  -> 错误处理、超时、metadata
  -> 拦截器、鉴权、日志、链路追踪
  -> 服务发现、负载均衡、健康检查
  -> 测试、压测、性能调优
  -> 网关、微服务项目落地
```

## 3. 第一阶段：Protocol Buffers 基础

建议时间：1 周

学习目标：

- 知道 gRPC、HTTP/2、Protobuf、代码生成之间的关系。
- 能独立编写 `.proto` 文件。
- 能生成 Go 消息代码和 gRPC 服务代码。

必学内容：

- `syntax = "proto3";`
- `package` 和 `option go_package`
- `message`
- 标量类型：`string`、`int32`、`int64`、`bool`、`double`
- `repeated`
- `map`
- `enum`
- 字段编号的作用
- 字段兼容性：不要复用已删除字段编号

练习：写一个 `user.proto`。

```proto
syntax = "proto3";

package user.v1;

option go_package = "example.com/grpc-demo/gen/user/v1;userv1";

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
}

message GetUserRequest {
  int64 id = 1;
}

message GetUserResponse {
  User user = 1;
}

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}
```

生成代码：

```bash
protoc --go_out=. --go_opt=paths=source_relative \
  --go-grpc_out=. --go-grpc_opt=paths=source_relative \
  user.proto
```

PowerShell 写法：

```powershell
protoc --go_out=. --go_opt=paths=source_relative `
  --go-grpc_out=. --go-grpc_opt=paths=source_relative `
  user.proto
```

阶段产出：

- 一个可以成功生成 `.pb.go` 和 `_grpc.pb.go` 的 proto 文件。
- 能解释生成代码里哪些是消息类型，哪些是 client/server 接口。

## 4. 第二阶段：跑通第一个 gRPC 服务

建议时间：1 周

学习目标：

- 能写一个 gRPC server。
- 能写一个 gRPC client。
- 能理解一次 RPC 调用的完整流程。

核心知识：

- `grpc.NewServer()`
- 注册服务：`RegisterXxxServer`
- 实现生成的 server interface
- client stub 的作用
- `context.WithTimeout`
- 长连接复用，避免每次请求都创建连接

练习项目：用户查询服务。

```text
grpc-user-demo/
  proto/user/v1/user.proto
  gen/user/v1/
  server/main.go
  client/main.go
  go.mod
```

功能：

- server 监听 `:50051`。
- client 调用 `GetUser`。
- server 返回内存里的用户数据。
- client 设置 3 秒超时。

阶段验收：

- 能在一个终端启动 server。
- 能在另一个终端运行 client 并得到返回结果。
- 能说清楚：proto -> 生成代码 -> 注册服务 -> client stub -> RPC 调用。

## 5. 第三阶段：掌握四种 RPC 模式

建议时间：1-2 周

| 模式 | 说明 | 常见场景 |
| --- | --- | --- |
| Unary RPC | 一次请求，一次响应 | 查询用户、创建订单 |
| Server Streaming | 一次请求，多次响应 | 导出数据、服务端推送进度 |
| Client Streaming | 多次请求，一次响应 | 上传文件、批量导入 |
| Bidirectional Streaming | 双向多次收发 | 聊天、实时协作、长连接通道 |

练习：在 `OrderService` 中实现：

- `GetOrder`：Unary。
- `ListOrders`：Server Streaming，按条件逐条返回订单。
- `CreateOrders`：Client Streaming，客户端连续发送订单，服务端返回汇总结果。
- `ChatOrderStatus`：Bidirectional Streaming，模拟订单状态实时沟通。

必须理解：

- streaming 方法为什么需要循环 `Recv()`？
- 服务端什么时候返回 `io.EOF`？
- 如何处理客户端提前取消？
- 如何避免发送速度过快导致阻塞？

## 6. 第四阶段：错误处理、超时、取消和 Metadata

建议时间：1 周

学习目标：

- 不再随便返回普通 `error`，而是使用标准 gRPC 状态码。
- 能正确设置 deadline 和 timeout。
- 能通过 metadata 传递 token、trace id、租户 id 等上下文信息。

核心知识：

- `status.Error`
- `codes.NotFound`
- `codes.InvalidArgument`
- `codes.Unauthenticated`
- `codes.PermissionDenied`
- `codes.Internal`
- `status.FromError`
- deadline 和 cancellation
- incoming metadata 和 outgoing metadata
- trailer metadata

练习：

- 用户不存在时返回 `codes.NotFound`。
- 参数非法时返回 `codes.InvalidArgument`。
- client 设置超时，server 模拟慢查询。
- client 通过 metadata 传递 `authorization`。
- server 从 metadata 读取 token。

阶段验收：

- client 能根据不同 gRPC code 做不同处理。
- server 能感知 client 取消请求。
- 能解释 timeout 和 deadline 的区别。

## 7. 第五阶段：拦截器和工程化中间件

建议时间：1 周

学习目标：

- 理解 gRPC interceptor 类似 HTTP middleware。
- 能统一处理日志、鉴权、panic recovery、trace id、耗时统计。

核心知识：

- Unary Server Interceptor
- Stream Server Interceptor
- Unary Client Interceptor
- Stream Client Interceptor
- interceptor chain
- panic recovery
- 统一错误映射

练习：

- `LoggingInterceptor`：记录 method、耗时、code。
- `AuthInterceptor`：校验 metadata 里的 token。
- `RecoveryInterceptor`：捕获 panic 并返回 `codes.Internal`。
- `TraceInterceptor`：没有 trace id 时生成一个，有则继续传递。

思考：

- 哪些逻辑应该放在业务 handler？
- 哪些逻辑应该放在 interceptor？
- server interceptor 和 client interceptor 各自适合做什么？

## 8. 第六阶段：安全、鉴权和传输加密

建议时间：1 周

学习目标：

- 理解 TLS 和 mTLS。
- 能实现 token 鉴权。
- 知道服务间通信如何保护身份和链路。

核心知识：

- insecure connection 只适合本地开发。
- TLS 保护传输链路。
- mTLS 可以同时验证 client 和 server 身份。
- metadata 适合传递 token，但不要把敏感信息写进日志。
- 鉴权失败常用 `codes.Unauthenticated`。
- 权限不足常用 `codes.PermissionDenied`。

练习：

- 给 server 配置 TLS 证书。
- client 使用 TLS 连接。
- 使用 interceptor 验证 Bearer Token。
- 模拟不同角色访问不同 RPC 方法。

## 9. 第七阶段：服务治理和生产能力

建议时间：1-2 周

学习目标：

- 让 gRPC 服务具备生产环境基本能力。
- 理解服务发现、负载均衡、健康检查和优雅关闭。

核心知识：

- Health Checking Protocol
- Server Reflection
- Graceful Shutdown
- Keepalive
- Retry 和 Request Hedging
- Load Balancing
- Name Resolver
- Service Config
- OpenTelemetry Metrics

练习：

- 增加健康检查接口。
- 开启 reflection，方便 `grpcurl` 调试。
- 实现优雅关闭，收到退出信号后停止接收新请求。
- 配置 client 重试。
- 接入 OpenTelemetry 指标采集。

推荐工具：

- `grpcurl`：命令行调试 gRPC 服务。
- `buf`：管理 proto、lint、breaking change 检查。
- `evans`：交互式 gRPC client。
- `ghz`：gRPC 压测工具。

## 10. 第八阶段：gRPC-Gateway 和对外 API

建议时间：1 周

学习目标：

- 理解内部服务用 gRPC，对外暴露 HTTP/JSON 的常见架构。
- 能使用 gRPC-Gateway 把 HTTP 请求转成 gRPC 调用。

核心知识：

- 浏览器和第三方系统通常更适合 HTTP/JSON。
- gRPC-Gateway 的作用。
- REST 路由和 RPC 方法的映射。
- OpenAPI 文档生成。
- 错误码映射。

练习：

- `GET /v1/users/{id}` -> `GetUser`
- `POST /v1/users` -> `CreateUser`
- HTTP server 和 gRPC server 同进程或分进程运行

## 11. 第九阶段：测试、压测和性能优化

建议时间：1 周

测试重点：

- handler 业务逻辑单元测试。
- 使用 bufconn 做内存级 gRPC 测试。
- metadata、deadline、错误码测试。
- streaming 场景测试。
- interceptor 测试。

性能重点：

- 避免每次请求重复创建连接。
- 合理设置 deadline。
- 避免超大 message。
- streaming 场景注意背压和内存占用。
- 观察 P50、P95、P99 延迟。
- 关注 QPS、错误率、连接数、goroutine 数量。

## 12. 综合项目：订单微服务系统

建议时间：2-3 周

服务拆分：

```text
api-gateway
  -> user-service
  -> product-service
  -> order-service
  -> notification-service
```

功能设计：

- 用户服务：注册、查询用户。
- 商品服务：查询商品、扣减库存。
- 订单服务：创建订单、查询订单、订单状态流式推送。
- 通知服务：通过 server streaming 模拟通知推送。
- API 网关：HTTP/JSON 对外，内部调用 gRPC。

技术要求：

- Protobuf 管理所有服务契约。
- 每个服务都有独立 server。
- 使用 interceptor 实现日志、鉴权、trace id。
- 使用 deadline 控制跨服务调用。
- 使用 gRPC status code 统一错误。
- 使用 health check 和 reflection。
- 至少写 10 个测试。
- 使用 `grpcurl` 完成接口调试。

推荐目录：

```text
grpc-shop/
  api/
    proto/
      user/v1/user.proto
      product/v1/product.proto
      order/v1/order.proto
  gen/
  cmd/
    user-service/
    product-service/
    order-service/
    api-gateway/
  internal/
    user/
    product/
    order/
    middleware/
    config/
  pkg/
    errors/
    logger/
  deployments/
    docker-compose.yml
  go.mod
```

## 13. 每周学习计划

| 周数 | 主题 | 产出 |
| --- | --- | --- |
| 第 1 周 | Protobuf 和代码生成 | `user.proto`，生成 Go 代码 |
| 第 2 周 | Unary RPC | 用户查询 demo |
| 第 3 周 | 四种 RPC 模式 | 订单服务 demo |
| 第 4 周 | 错误、超时、metadata | 标准错误处理和 token 传递 |
| 第 5 周 | 拦截器 | 日志、鉴权、恢复、trace |
| 第 6 周 | TLS、mTLS、鉴权 | 安全连接 demo |
| 第 7 周 | 健康检查、反射、优雅关闭 | 生产化 server |
| 第 8 周 | gRPC-Gateway | HTTP/JSON 网关 |
| 第 9 周 | 测试和压测 | bufconn 测试、ghz 压测 |
| 第 10 周 | 综合项目 | 订单微服务系统 |

## 14. 高频面试问题

基础问题：

- gRPC 和 REST 有什么区别？
- gRPC 为什么通常比 JSON/HTTP 更高效？
- Protobuf 字段编号有什么用？
- proto3 和 proto2 有什么区别？
- `option go_package` 是干什么的？

Go 实战问题：

- gRPC server 是如何注册服务的？
- 生成的 `_grpc.pb.go` 里包含什么？
- client 连接是否应该每次请求都新建？
- 如何设置超时？
- 如何处理客户端取消？

生产问题：

- 如何做鉴权？
- 如何做日志和链路追踪？
- 如何做健康检查？
- 如何做优雅关闭？
- 如何处理跨服务错误码？
- 如何设计 proto 的兼容性？
- streaming 场景如何避免内存问题？

## 15. 学习资源

优先看官方文档，再看博客和视频。

- [gRPC Go Quick Start](https://grpc.io/docs/languages/go/quickstart/)
- [gRPC Go Basics Tutorial](https://grpc.io/docs/languages/go/basics/)
- [gRPC Core Concepts](https://grpc.io/docs/what-is-grpc/core-concepts/)
- [gRPC Guides](https://grpc.io/docs/guides/)
- [Protocol Buffers Go Tutorial](https://protobuf.dev/getting-started/gotutorial/)
- [Protocol Buffers Language Guide](https://protobuf.dev/programming-guides/proto3/)
- [Go Tutorials](https://go.dev/doc/tutorial/)
- [grpc-go GitHub](https://github.com/grpc/grpc-go)
- [grpc-gateway GitHub](https://github.com/grpc-ecosystem/grpc-gateway)
- [buf 官方文档](https://buf.build/docs/)

## 16. 最小可行学习闭环

如果你想先快速建立信心，可以按这个闭环走：

1. 写一个 `hello.proto`。
2. 用 `protoc` 生成 Go 代码。
3. 写 server。
4. 写 client。
5. 加 timeout。
6. 加 metadata token。
7. 加 interceptor 日志。
8. 用 `grpcurl` 调试。
9. 写一个 bufconn 测试。
10. 把 demo 改造成一个小型 user-service。

完成这个闭环后，再进入 streaming、TLS、gRPC-Gateway、服务治理和综合项目。
