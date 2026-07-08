# gRPC 面试题与参考答案

> 适用对象：准备 Go 后端工程师面试、已经系统学习过 gRPC 基础教程、希望把知识转化为面试表达的人。
>
> 使用方式：先按模块复习，再尝试不看答案口述。面试时不要死背原文，要抓住“原理 + Go 实现 + 生产实践 + 踩坑经验”四个层次。

---

## 一、面试复习总览

gRPC 面试通常不会只问“会不会写 proto”。比较完整的考察路径是：

```text
RPC 基础
  -> gRPC 和 HTTP/2 原理
  -> Protobuf 契约设计
  -> Go 代码生成与服务实现
  -> 四种 RPC 模式
  -> context、超时、取消
  -> status code、metadata、错误模型
  -> interceptor、鉴权、日志、恢复
  -> TLS、mTLS、安全
  -> 服务发现、负载均衡、健康检查、反射
  -> gRPC-Gateway
  -> 测试、压测、性能优化
  -> 微服务项目落地与故障排查
```

回答问题时建议使用这个结构：

```text
1. 先给定义。
2. 再说它解决什么问题。
3. 再结合 Go gRPC 怎么实现。
4. 最后补充生产实践中的注意点。
```

例如回答“什么是 interceptor”时，不要只说“类似中间件”。更好的回答是：

```text
Interceptor 是 gRPC 的拦截器机制，可以在请求进入业务 handler 前后统一处理日志、鉴权、恢复 panic、metrics、trace 等横切逻辑。
Go 里常见的是 UnaryServerInterceptor 和 StreamServerInterceptor。
生产中一般会用 ChainUnaryInterceptor 组合多个拦截器，并注意顺序，例如 recovery 放在外层，日志和 trace 记录 method、code、duration。
```

---

## 二、RPC 与 gRPC 基础

### 1. 什么是 RPC？它和本地函数调用有什么区别？

RPC 是 Remote Procedure Call，远程过程调用。它希望让开发者像调用本地函数一样调用远程服务。

本地函数调用只涉及进程内的栈、参数和返回值；RPC 调用跨进程、跨机器，需要处理网络连接、序列化、反序列化、超时、重试、错误码、服务发现、安全认证等问题。

面试回答要点：

- RPC 是一种调用模型，不是某一个具体框架。
- gRPC、Dubbo、Thrift 都是 RPC 框架。
- RPC 需要把请求对象序列化成字节，通过网络发送到远端，再由远端反序列化并执行。
- RPC 的难点不是“像函数一样调用”，而是网络不可靠带来的超时、失败、重试和一致性问题。

生产补充：

远程调用不能真的当成本地函数调用。远程调用可能慢、可能失败、可能重复执行、可能部分成功，所以必须设计超时、幂等、错误码和降级策略。

---

### 2. gRPC 是什么？

gRPC 是 Google 开源的高性能 RPC 框架，默认使用 HTTP/2 作为传输协议，使用 Protocol Buffers 作为接口定义语言和序列化格式。

gRPC 的核心组成：

- `.proto` 文件：定义 message 和 service。
- Protobuf：负责数据结构定义和二进制序列化。
- HTTP/2：提供多路复用、流式传输、header 压缩等能力。
- 代码生成：生成不同语言的 client 和 server stub。
- runtime：处理连接、编码、解码、拦截器、metadata、deadline、status code 等。

适合场景：

- 内部微服务调用。
- 多语言服务之间通信。
- 低延迟、高吞吐场景。
- 需要强契约的服务接口。
- 流式传输场景。

不适合或需要权衡的场景：

- 浏览器直接调用比较麻烦，通常需要 gRPC-Web 或 Gateway。
- 面向公开开放平台时，HTTP/JSON 生态更通用。
- 排查问题时二进制协议不如纯 JSON 直观，需要 grpcurl、reflection、日志和 tracing 支持。

---

### 3. gRPC 和 REST/HTTP JSON 有什么区别？

可以从协议、契约、序列化、性能、生态和适用场景回答。

| 对比点 | gRPC | REST/HTTP JSON |
| --- | --- | --- |
| 传输协议 | HTTP/2 | 通常 HTTP/1.1 或 HTTP/2 |
| 接口契约 | proto 强契约 | OpenAPI 或约定 |
| 数据格式 | Protobuf 二进制 | JSON 文本 |
| 性能 | 通常更高效 | 可读性好但体积较大 |
| 流式能力 | 原生支持四种 RPC 模式 | 需要 SSE/WebSocket/Chunked |
| 浏览器支持 | 原生较弱 | 原生友好 |
| 调试方式 | grpcurl、reflection | curl、浏览器、Postman |
| 适用场景 | 内部服务、高性能、多语言 | 对外 API、前端接口、开放平台 |

回答时不要绝对说 gRPC 一定比 REST 好。更稳妥的说法是：

gRPC 更适合内部服务之间的强契约、高性能调用；REST/HTTP JSON 更适合对外开放和浏览器生态。实际项目里经常是内部 gRPC，对外通过 API Gateway 暴露 HTTP/JSON。

---

### 4. gRPC 为什么选择 HTTP/2？

HTTP/2 给 gRPC 提供了几个关键能力：

- 多路复用：一个 TCP 连接上可以并发多个 stream，减少连接数量。
- 双向流：天然支持 client streaming、server streaming、bidirectional streaming。
- Header 压缩：使用 HPACK 压缩 header，降低重复 metadata 成本。
- 二进制分帧：请求和响应被拆成 frame，更适合流式协议。
- 长连接：适合微服务之间频繁调用。

面试追问：HTTP/2 多路复用是否解决了所有队头阻塞？

参考答案：

HTTP/2 解决的是 HTTP/1.1 层面的应用层队头阻塞，一个 TCP 连接可以同时承载多个 stream。但 TCP 层仍然存在队头阻塞，如果底层丢包，整个 TCP 连接上的数据都会受到影响。因此高并发场景下仍要关注连接数、网络质量和负载均衡策略。

---

### 5. gRPC 的一次调用大致经历了哪些步骤？

一次 Unary RPC 调用大致流程：

```text
客户端调用生成的 client 方法
  -> 请求 message 被 Protobuf 序列化
  -> gRPC runtime 添加 method、metadata、deadline 等信息
  -> 通过 HTTP/2 stream 发送到服务端
  -> 服务端反序列化请求
  -> 执行 server interceptor
  -> 执行业务 handler
  -> 返回 response 或 error status
  -> 客户端反序列化 response 或解析 status error
```

Go 里对应的代码通常是：

```go
conn, err := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
if err != nil {
	return err
}
defer conn.Close()

client := userv1.NewUserServiceClient(conn)
resp, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: 1})
```

服务端：

```go
server := grpc.NewServer()
userv1.RegisterUserServiceServer(server, userService)
server.Serve(listener)
```

---

## 三、Protobuf 与契约设计

### 6. Protobuf 是什么？为什么 gRPC 常用 Protobuf？

Protobuf 是一种接口描述语言和二进制序列化协议。它可以用 `.proto` 文件定义数据结构和服务接口，然后生成多种语言的代码。

gRPC 使用 Protobuf 的优势：

- 强契约：服务接口、请求、响应都有明确类型。
- 跨语言：可以生成 Go、Java、Python、Node 等代码。
- 性能较好：二进制体积小，序列化和反序列化效率高。
- 向前兼容：通过字段编号支持接口演进。
- 与 gRPC 代码生成天然结合。

注意点：

Protobuf 的可读性不如 JSON，需要工具支持调试。接口设计一旦发布，要谨慎修改字段编号和语义。

---

### 7. proto3 中字段编号有什么作用？为什么不能随便改？

字段编号是 Protobuf 二进制编码中的关键标识。序列化时使用字段编号，而不是字段名。

例如：

```proto
message User {
  int64 id = 1;
  string name = 2;
}
```

`1` 和 `2` 才是二进制编码中真正用于识别字段的 tag。字段名更多是给代码生成和人阅读用。

不能随便改字段编号的原因：

- 老客户端和新服务端之间靠字段编号识别数据。
- 如果把 `name = 2` 改成 `email = 2`，老数据可能被错误解释。
- 删除字段后也不要复用编号，否则会产生兼容性问题。

推荐做法：

```proto
message User {
  int64 id = 1;
  reserved 2;
  reserved "old_name";
  string email = 3;
}
```

使用 `reserved` 保留删除的字段编号和字段名，避免未来误用。

---

### 8. proto3 的默认值有什么坑？

proto3 中标量类型都有默认值：

- string 默认 `""`
- int32/int64 默认 `0`
- bool 默认 `false`
- enum 默认第一个枚举值

问题在于：默认值无法直接区分“客户端没有传”和“客户端传了零值”。

例如：

```proto
message UpdateUserRequest {
  string name = 1;
  int32 age = 2;
}
```

如果 `age = 0`，服务端不知道客户端是想把 age 改成 0，还是没有传 age。

解决方式：

- 使用 `optional` 字段。
- 使用 wrapper 类型。
- 使用 field mask。
- 设计明确的 Update API，不要滥用零值表达业务语义。

示例：

```proto
message UpdateUserRequest {
  int64 id = 1;
  optional string name = 2;
  optional int32 age = 3;
}
```

---

### 9. proto 中 `package` 和 `go_package` 有什么区别？

`package` 是 Protobuf 层面的命名空间，用于避免 proto message 和 service 名称冲突。

`go_package` 是 Go 代码生成时使用的 import 路径和包名。

示例：

```proto
syntax = "proto3";

package user.v1;

option go_package = "example.com/grpc-shop/gen/user/v1;userv1";
```

含义：

- `user.v1` 是 proto 包名，gRPC full method 会包含它，例如 `/user.v1.UserService/GetUser`。
- `example.com/grpc-shop/gen/user/v1` 是 Go import 路径。
- `userv1` 是生成的 Go package 名。

常见错误：

- 忘记写 `go_package`，导致生成路径混乱。
- proto package 和 Go package 混为一谈。
- 多个 proto 文件生成到同一个 Go 包时命名冲突。

---

### 10. enum 在 proto3 中为什么建议第一个值是 UNSPECIFIED？

proto3 中 enum 的默认值是第一个枚举值，且必须是 0。

如果第一个值直接写成业务状态：

```proto
enum OrderStatus {
  CREATED = 0;
  PAID = 1;
}
```

那么客户端没传状态时，服务端会看到 `CREATED`，无法区分“未指定”和“真的创建状态”。

推荐：

```proto
enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_CREATED = 1;
  ORDER_STATUS_PAID = 2;
}
```

这样零值表示未知或未指定，业务状态从 1 开始，语义更清晰。

---

### 11. `repeated` 和 `map` 分别适合什么场景？

`repeated` 表示列表：

```proto
message ListUsersResponse {
  repeated User users = 1;
}
```

适合有顺序、可重复、列表展示的场景。

`map` 表示键值对：

```proto
message BatchGetUsersResponse {
  map<int64, User> users = 1;
}
```

适合通过 key 快速定位 value 的场景。

注意：

- `map` 的遍历顺序不要依赖。
- 对外 API 中如果顺序很重要，优先使用 `repeated`。
- `map` 的 key 类型有限，不能使用 message 作为 key。

---

### 12. oneof 有什么用？

`oneof` 表示一组字段中最多只能设置一个，适合表达互斥结构。

示例：

```proto
message PaymentMethod {
  oneof method {
    CreditCard credit_card = 1;
    WeChatPay wechat_pay = 2;
    AliPay ali_pay = 3;
  }
}
```

适用场景：

- 多种支付方式只能选一种。
- 搜索条件中多种查询方式互斥。
- 事件消息中不同事件类型对应不同 payload。

注意：

`oneof` 适合协议层表达互斥，但业务校验仍然要在服务端做，不能完全依赖客户端生成代码。

---

### 13. 如何设计一个可演进的 proto 契约？

要点：

- 字段编号发布后不要修改含义。
- 删除字段时使用 `reserved`。
- 新增字段要兼容老客户端，不能要求老客户端必须传。
- enum 第一个值使用 `UNSPECIFIED`。
- Request 和 Response 分开定义，不要直接暴露内部数据库模型。
- 包名带版本，例如 `user.v1`。
- 金额使用整数最小单位，例如 `price_cents int64`。
- 时间可以使用 `google.protobuf.Timestamp`，也可以统一使用毫秒时间戳，但要有团队规范。

示例：

```proto
message CreateOrderRequest {
  int64 user_id = 1;
  int64 product_id = 2;
  int32 quantity = 3;
  string request_id = 4;
}
```

其中 `request_id` 可以用于幂等，后续实现重试时很有价值。

---

## 四、Go gRPC 实现

### 14. Go gRPC 服务端一般怎么写？

典型步骤：

1. 编写 proto。
2. 生成 Go 代码。
3. 定义结构体实现生成的 Server 接口。
4. 创建 `grpc.Server`。
5. 注册服务。
6. 监听端口并 Serve。

示例：

```go
type userServer struct {
	userv1.UnimplementedUserServiceServer
}

func (s *userServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
	return &userv1.GetUserResponse{
		User: &userv1.User{Id: req.GetId(), Name: "alice"},
	}, nil
}

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatal(err)
	}

	server := grpc.NewServer()
	userv1.RegisterUserServiceServer(server, &userServer{})

	if err := server.Serve(lis); err != nil {
		log.Fatal(err)
	}
}
```

面试注意：

`UnimplementedUserServiceServer` 建议嵌入，这样 proto 新增方法时服务端有默认未实现行为，兼容性更好。

---

### 15. Go gRPC 客户端一般怎么写？

典型步骤：

1. 建立连接。
2. 创建生成的 client。
3. 创建 context，设置 timeout。
4. 调用 RPC 方法。
5. 处理 response 和 error。

示例：

```go
conn, err := grpc.Dial(
	"localhost:50051",
	grpc.WithTransportCredentials(insecure.NewCredentials()),
)
if err != nil {
	return err
}
defer conn.Close()

client := userv1.NewUserServiceClient(conn)

ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()

resp, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: 1})
if err != nil {
	return err
}

fmt.Println(resp.GetUser().GetName())
```

生产注意：

- 不要每次请求都重新 `Dial`。
- gRPC `ClientConn` 是并发安全的，可以复用。
- 每次 RPC 调用要设置合理 timeout。
- 连接生命周期通常由服务初始化和关闭流程管理。

---

### 16. 为什么不建议每次请求都新建 gRPC 连接？

因为建立连接有成本，包括 TCP 握手、TLS 握手、HTTP/2 初始化、连接池和负载均衡状态维护。

如果每次请求都 Dial：

- 延迟更高。
- 资源消耗更大。
- 容易造成连接风暴。
- 负载均衡和连接状态不稳定。

推荐做法：

- 服务启动时建立下游连接。
- 使用同一个 `ClientConn` 创建 client 并复用。
- 进程退出时关闭连接。

Go 中 `ClientConn` 是并发安全的，多个 goroutine 可以共享。

---

### 17. `UnimplementedXXXServer` 有什么作用？

生成的 gRPC 代码通常会包含 `UnimplementedXXXServer`。服务实现结构体嵌入它可以获得默认方法实现。

作用：

- 当 proto 新增 RPC 方法时，旧服务端仍能编译。
- 未实现的方法会返回 `Unimplemented`。
- 有利于接口向前演进。

示例：

```go
type userServer struct {
	userv1.UnimplementedUserServiceServer
}
```

如果不嵌入，当 proto 新增方法后，Go 接口变化可能导致编译失败。

面试补充：

这不是让你忽略新方法，而是让服务升级过程更平滑。真正上线前仍然应该明确实现或确认不支持的方法。

---

## 五、四种 RPC 模式

### 18. gRPC 有哪四种 RPC 模式？

四种模式：

1. Unary RPC：一请求一响应。
2. Server Streaming：客户端一个请求，服务端返回多个响应。
3. Client Streaming：客户端发送多个请求，服务端返回一个响应。
4. Bidirectional Streaming：客户端和服务端都可以持续发送消息。

proto 示例：

```proto
service OrderService {
  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
  rpc WatchOrderStatus(WatchOrderStatusRequest) returns (stream OrderStatusEvent);
  rpc UploadOrders(stream UploadOrderRequest) returns (UploadOrderResponse);
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}
```

选择建议：

- 普通查询/创建：Unary。
- 状态推送/日志订阅：Server Streaming。
- 批量上传/客户端持续上报：Client Streaming。
- 聊天、实时协作、双向同步：Bidirectional Streaming。

---

### 19. Server Streaming 如何实现？适合什么场景？

Server Streaming 是客户端发送一个请求，服务端连续返回多个消息。

服务端 Go 示例：

```go
func (s *orderServer) WatchOrderStatus(
	req *orderv1.WatchOrderStatusRequest,
	stream orderv1.OrderService_WatchOrderStatusServer,
) error {
	events := []*orderv1.OrderStatusEvent{
		{OrderId: req.GetOrderId(), Message: "created"},
		{OrderId: req.GetOrderId(), Message: "paid"},
		{OrderId: req.GetOrderId(), Message: "shipped"},
	}

	for _, event := range events {
		select {
		case <-stream.Context().Done():
			return stream.Context().Err()
		default:
		}

		if err := stream.Send(event); err != nil {
			return err
		}
	}

	return nil
}
```

适合场景：

- 订单状态推送。
- 任务进度推送。
- 日志流。
- 服务端持续返回分页或批量结果。

注意：

- 要处理客户端取消。
- 不要无限制缓存消息。
- 要考虑发送速度和客户端消费速度。

---

### 20. Client Streaming 如何实现？

Client Streaming 是客户端连续发送多个请求，服务端最后返回一个响应。

服务端核心是循环 `Recv()`：

```go
func (s *orderServer) CreateOrders(stream orderv1.OrderService_CreateOrdersServer) error {
	var count int32

	for {
		req, err := stream.Recv()
		if errors.Is(err, io.EOF) {
			return stream.SendAndClose(&orderv1.CreateOrdersResponse{Count: count})
		}
		if err != nil {
			return err
		}

		if req.GetUserId() <= 0 {
			return status.Error(codes.InvalidArgument, "user_id is required")
		}

		count++
	}
}
```

适合场景：

- 批量上传。
- 客户端持续采集指标后一次性汇总。
- 上传大文件分片。

注意：

- `io.EOF` 表示客户端发送结束。
- 服务端要限制消息大小和总数量。
- 中途错误要明确返回 status code。

---

### 21. 双向流式 RPC 容易遇到什么问题？

双向流最灵活，也最复杂。

常见问题：

- 读写 goroutine 生命周期管理不清。
- 客户端或服务端取消后 goroutine 泄漏。
- 发送方太快，接收方太慢，导致背压问题。
- 错误处理复杂，不知道该关闭发送还是关闭整个 stream。
- 多个客户端之间消息隔离不清。

推荐答法：

双向流适合实时通信，但要谨慎设计协议。通常会把读和写拆到不同 goroutine，通过 context、channel 和 errgroup 管理生命周期。任何一侧出错或取消，都要能通知另一侧退出。

---

### 22. 流式 RPC 中如何处理客户端取消？

服务端可以通过 `stream.Context().Done()` 感知取消。

示例：

```go
select {
case <-stream.Context().Done():
	return stream.Context().Err()
case msg := <-eventCh:
	return stream.Send(msg)
}
```

还要注意：

- `Send` 或 `Recv` 返回错误时也要退出。
- 不要在后台 goroutine 中继续发送。
- 如果使用 channel，要设计关闭机制。
- 长时间流式连接要有心跳或超时策略。

面试表达：

流式 RPC 的关键不是把 `Send` 写出来，而是正确处理生命周期，避免客户端断开后服务端 goroutine 泄漏。

---

## 六、context、超时与取消

### 23. gRPC 中为什么必须设置 timeout？

因为远程调用可能因为网络、下游服务、队列阻塞、数据库慢查询等原因长时间不返回。如果不设置 timeout：

- goroutine 长时间占用。
- 连接和内存资源泄漏。
- 上游请求被拖死。
- 故障可能级联扩散。

Go 客户端通常这样写：

```go
ctx, cancel := context.WithTimeout(context.Background(), 800*time.Millisecond)
defer cancel()

resp, err := client.GetUser(ctx, req)
```

服务端也应该尊重 ctx：

```go
select {
case <-ctx.Done():
	return nil, ctx.Err()
case result := <-work:
	return result, nil
}
```

---

### 24. deadline 和 timeout 有什么区别？

timeout 是相对时间，例如 800ms 后超时。

deadline 是绝对时间点，例如 2026-07-05 20:00:00 前必须完成。

Go 中：

```go
context.WithTimeout(parent, 800*time.Millisecond)
context.WithDeadline(parent, time.Now().Add(800*time.Millisecond))
```

在 gRPC 里，客户端设置 deadline 后，会通过请求上下文传给服务端。服务端可以通过 `ctx.Deadline()` 获取剩余时间，也可以通过 `ctx.Done()` 感知取消。

---

### 25. 如何设计跨服务调用的超时预算？

假设外部请求总超时是 2 秒，OrderService 需要调用 UserService、ProductService、PaymentService。

不能每个下游都设置 2 秒，否则整体可能超过上游预算。

建议：

```text
总预算：2s
网关处理：100ms
OrderService 自身逻辑：200ms
UserService：300ms
ProductService：300ms
PaymentService：800ms
预留：300ms
```

实现上可以使用父 ctx 派生子 ctx：

```go
userCtx, cancel := context.WithTimeout(ctx, 300*time.Millisecond)
defer cancel()

_, err := userClient.GetUser(userCtx, req)
```

面试亮点：

超时不是随便写一个数字，而是要根据调用链路做预算分配，并保留一定余量。

---

## 七、错误处理与 status code

### 26. gRPC 错误模型是什么？

gRPC 使用 status 表达错误，包括：

- code：机器可读的错误码。
- message：人类可读的错误描述。
- details：可选的结构化错误详情。

Go 中返回错误：

```go
return nil, status.Error(codes.NotFound, "user not found")
```

客户端解析：

```go
st, ok := status.FromError(err)
if ok {
	fmt.Println(st.Code(), st.Message())
}
```

常见 code：

- `InvalidArgument`：参数错误。
- `NotFound`：资源不存在。
- `AlreadyExists`：资源已存在。
- `PermissionDenied`：权限不足。
- `Unauthenticated`：未认证。
- `FailedPrecondition`：前置条件不满足。
- `Unavailable`：服务不可用。
- `DeadlineExceeded`：超时。
- `Internal`：内部错误。
- `Unimplemented`：未实现。

---

### 27. `InvalidArgument` 和 `FailedPrecondition` 怎么区分？

`InvalidArgument` 表示请求参数本身不合法，不依赖系统当前状态。

例如：

- id <= 0
- email 格式错误
- quantity <= 0

`FailedPrecondition` 表示参数本身可能合法，但当前业务状态不允许执行。

例如：

- 库存不足
- 订单已取消，不能支付
- 用户状态被冻结，不能下单

回答示例：

创建订单时 `quantity = -1` 应返回 `InvalidArgument`；`quantity = 2` 但库存只有 1，应返回 `FailedPrecondition`。

---

### 28. `Unauthenticated` 和 `PermissionDenied` 怎么区分？

`Unauthenticated` 表示身份未认证或认证失败。

例如：

- 没有 token。
- token 过期。
- token 格式错误。

`PermissionDenied` 表示身份已经确认，但没有权限访问资源。

例如：

- 普通用户访问管理员接口。
- 用户 A 查询用户 B 的私有订单。

简洁回答：

先看“你是谁”是否确认。没确认是 `Unauthenticated`；确认了但不允许是 `PermissionDenied`。

---

### 29. 为什么不能所有错误都返回 Internal？

因为调用方无法根据错误类型做正确处理。

例如：

- `NotFound`：前端可以显示资源不存在。
- `InvalidArgument`：客户端应该修正参数。
- `Unavailable`：可以考虑重试。
- `FailedPrecondition`：提示业务状态不允许。
- `Internal`：通常表示服务端未知故障，需要报警排查。

如果全部返回 `Internal`：

- 客户端无法区分可重试和不可重试。
- 监控无法区分业务错误和系统故障。
- 排查问题困难。
- 对外错误体验差。

---

### 30. 如何在 gRPC 中返回结构化错误详情？

可以使用 `status.WithDetails` 搭配 Google RPC errdetails。

示例：

```go
st := status.New(codes.InvalidArgument, "invalid request")
st, err := st.WithDetails(&errdetails.BadRequest{
	FieldViolations: []*errdetails.BadRequest_FieldViolation{
		{
			Field:       "email",
			Description: "email format is invalid",
		},
	},
})
if err != nil {
	return nil, status.Error(codes.Internal, "build error detail failed")
}
return nil, st.Err()
```

适用场景：

- 表单字段校验错误。
- 多字段错误同时返回。
- 对外 API 需要统一错误格式。

注意：

结构化 details 虽然强大，但要控制复杂度，不要把内部敏感信息暴露给调用方。

---

## 八、Metadata、Header 与 Trailer

### 31. gRPC metadata 是什么？

metadata 是 gRPC 中附加在 RPC 调用上的键值对，类似 HTTP header。

常见用途：

- 传递 token。
- 传递 trace id。
- 传递租户 ID。
- 传递语言、地区、客户端版本。
- 返回 quota、限流、调试信息。

客户端发送：

```go
md := metadata.Pairs("authorization", "Bearer token", "x-trace-id", "trace-001")
ctx := metadata.NewOutgoingContext(context.Background(), md)
resp, err := client.GetUser(ctx, req)
```

服务端读取：

```go
md, ok := metadata.FromIncomingContext(ctx)
if ok {
	values := md.Get("authorization")
}
```

注意：

metadata 的 key 通常小写。不要在 metadata 中放大对象或敏感明文信息。

---

### 32. Header 和 Trailer 有什么区别？

Header 是响应开始时发送的 metadata。

Trailer 是响应结束时发送的 metadata。

用途：

- Header：返回请求开始阶段就能确定的信息，例如 request id、server version。
- Trailer：返回请求结束后才能确定的信息，例如处理耗时、quota 剩余、统计信息。

Go 服务端设置：

```go
grpc.SendHeader(ctx, metadata.Pairs("x-server", "user-service"))
grpc.SetTrailer(ctx, metadata.Pairs("x-cost-ms", "12"))
```

客户端接收：

```go
var header metadata.MD
var trailer metadata.MD

resp, err := client.GetUser(ctx, req, grpc.Header(&header), grpc.Trailer(&trailer))
```

---

## 九、Interceptor 拦截器

### 33. gRPC interceptor 是什么？

Interceptor 是 gRPC 的拦截器机制，用来处理横切逻辑。它类似 HTTP middleware，但作用在 gRPC 调用链上。

常见用途：

- 访问日志。
- 鉴权。
- panic recovery。
- metrics。
- tracing。
- 限流。
- 参数校验。
- 超时控制。

Unary Server Interceptor 示例：

```go
func LoggingInterceptor() grpc.UnaryServerInterceptor {
	return func(
		ctx context.Context,
		req any,
		info *grpc.UnaryServerInfo,
		handler grpc.UnaryHandler,
	) (any, error) {
		start := time.Now()
		resp, err := handler(ctx, req)
		log.Printf("method=%s code=%s duration=%s", info.FullMethod, status.Code(err), time.Since(start))
		return resp, err
	}
}
```

---

### 34. Unary interceptor 和 Stream interceptor 有什么区别？

Unary interceptor 处理一请求一响应的方法。

Stream interceptor 处理流式 RPC。流式 RPC 的请求和响应可能有多个，所以不能像 Unary 一样只围绕一个 req/resp 处理。

Unary：

```go
grpc.UnaryServerInterceptor(...)
```

Stream：

```go
grpc.StreamServerInterceptor(...)
```

Stream interceptor 通常要包装 `ServerStream`，重写 `RecvMsg` 和 `SendMsg`，才能在每次收发消息时做日志、校验或统计。

面试补充：

如果只配置 Unary interceptor，流式 RPC 不会自动被它覆盖。生产中要分别配置 unary 和 stream 的日志、鉴权、recovery。

---

### 35. 多个 interceptor 的顺序重要吗？

重要。

例如：

```go
grpc.ChainUnaryInterceptor(
	Recovery(),
	Tracing(),
	Logging(),
	Auth(),
)
```

调用顺序类似洋葱模型。外层 interceptor 先进入，后退出。

常见建议：

- Recovery 放外层，防止后续拦截器或业务 panic 导致进程异常。
- Tracing 尽早创建 span，让后续日志都能带 trace id。
- Logging 记录 method、code、duration。
- Auth 在业务 handler 前执行，避免未授权请求进入业务逻辑。

但具体顺序要看团队规范。关键是能解释为什么这样排。

---

### 36. 如何用 interceptor 做鉴权？

服务端从 metadata 中读取 token，校验后决定是否放行。

示例：

```go
func AuthInterceptor(publicMethods map[string]bool) grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
		if publicMethods[info.FullMethod] {
			return handler(ctx, req)
		}

		md, ok := metadata.FromIncomingContext(ctx)
		if !ok {
			return nil, status.Error(codes.Unauthenticated, "missing metadata")
		}

		tokens := md.Get("authorization")
		if len(tokens) == 0 || tokens[0] != "Bearer secret" {
			return nil, status.Error(codes.Unauthenticated, "invalid token")
		}

		return handler(ctx, req)
	}
}
```

生产注意：

- token 校验应对接真实认证服务或 JWT。
- 白名单方法要明确，例如 health check、reflection。
- 不要在日志中打印完整 token。
- 鉴权失败用 `Unauthenticated`，权限不足用 `PermissionDenied`。

---

### 37. 如何用 interceptor 做 panic recovery？

在 interceptor 中 `defer recover()`，将 panic 转成 `Internal` 错误，同时记录日志。

示例：

```go
func RecoveryInterceptor() grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp any, err error) {
		defer func() {
			if r := recover(); r != nil {
				log.Printf("panic method=%s value=%v", info.FullMethod, r)
				err = status.Error(codes.Internal, "internal server error")
			}
		}()

		return handler(ctx, req)
	}
}
```

注意：

Recovery 不是让你忽略 panic。它的作用是保护进程不崩溃，并把错误转成可观测信号。panic 仍然应该报警和修复。

---

## 十、安全、TLS 与 mTLS

### 38. gRPC 如何启用 TLS？

服务端加载证书和私钥：

```go
creds, err := credentials.NewServerTLSFromFile("server.crt", "server.key")
if err != nil {
	return err
}

server := grpc.NewServer(grpc.Creds(creds))
```

客户端使用 TLS credential：

```go
creds, err := credentials.NewClientTLSFromFile("ca.crt", "server.example.com")
if err != nil {
	return err
}

conn, err := grpc.Dial("server.example.com:50051", grpc.WithTransportCredentials(creds))
```

TLS 提供：

- 加密传输。
- 服务端身份验证。
- 防止中间人攻击。

学习阶段常见 `insecure.NewCredentials()`，但生产环境不能裸奔。

---

### 39. mTLS 是什么？和 TLS 有什么区别？

TLS 通常是客户端验证服务端证书。

mTLS 是 mutual TLS，双向 TLS，服务端也验证客户端证书。

适用场景：

- 内部微服务之间强身份认证。
- 零信任网络。
- 服务网格，例如 Istio、Linkerd。
- 高安全要求系统。

回答要点：

mTLS 不只是加密，还能确认“调用方服务是谁”。相比 token，mTLS 更偏传输层身份认证；实际项目中也可能同时使用 mTLS 和应用层 token。

---

### 40. TLS、Token、mTLS 分别解决什么问题？

| 机制 | 解决的问题 |
| --- | --- |
| TLS | 加密传输，验证服务端身份 |
| Token/JWT | 应用层用户或服务身份认证 |
| mTLS | 双向服务身份认证 |

组合方式：

- 对外 HTTP API：TLS + 用户 Token。
- 内部服务调用：mTLS + service identity。
- 需要用户上下文透传：mTLS 保护服务身份，metadata 传用户 token 或 user id。

---

## 十一、服务治理与生产能力

### 41. gRPC 健康检查有什么用？

健康检查用于告诉负载均衡器、服务发现系统或 Kubernetes：当前服务是否可接收流量。

gRPC 官方有 health checking protocol。

服务端注册：

```go
healthServer := health.NewServer()
grpc_health_v1.RegisterHealthServer(grpcServer, healthServer)
healthServer.SetServingStatus("", grpc_health_v1.HealthCheckResponse_SERVING)
```

客户端或运维检查：

```powershell
grpcurl -plaintext localhost:50051 grpc.health.v1.Health/Check
```

用途：

- Kubernetes readiness/liveness。
- 服务发现剔除异常实例。
- 上线发布前检查。

注意：

健康检查不应该只返回进程活着，还应该考虑关键依赖是否可用。不过也不能做得太重，否则健康检查本身会造成压力。

---

### 42. gRPC reflection 是什么？

Reflection 允许客户端在不知道 proto 文件的情况下查询服务暴露了哪些方法和 message。

服务端启用：

```go
reflection.Register(grpcServer)
```

启用后可以用 grpcurl 列出服务：

```powershell
grpcurl -plaintext localhost:50051 list
```

适合：

- 开发调试。
- 测试环境排查。
- 内部工具发现服务接口。

生产注意：

公网服务慎重开启 reflection，因为它会暴露服务接口信息。可以只在内网或测试环境开启。

---

### 43. gRPC 服务发现和负载均衡怎么做？

gRPC 客户端需要知道服务实例地址，并在多个实例之间选择目标。

常见方案：

- DNS：通过域名解析多个 IP。
- Consul/Etcd/Nacos：服务注册与发现。
- Kubernetes Service：通过 kube-proxy 或 sidecar。
- xDS/Envoy：更复杂的服务治理。
- 服务网格：Istio、Linkerd 等。

负载均衡策略：

- pick_first：默认常见策略，优先选择一个地址。
- round_robin：轮询多个后端。

Go 配置示例：

```go
conn, err := grpc.Dial(
	"dns:///user-service.default.svc.cluster.local:50051",
	grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`),
	grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

面试重点：

gRPC 的负载均衡通常发生在客户端侧，客户端 resolver 解析服务地址，balancer 选择具体连接。

---

### 44. gRPC 重试要注意什么？

重试可以提高短暂故障下的成功率，但也可能造成重复执行和流量放大。

适合重试：

- `Unavailable`
- 短暂网络错误
- 幂等查询

不适合直接重试：

- 创建订单
- 扣库存
- 支付
- 任何非幂等写操作

如果要重试写操作，必须设计幂等键，例如：

```proto
message CreateOrderRequest {
  int64 user_id = 1;
  int64 product_id = 2;
  int32 quantity = 3;
  string request_id = 4;
}
```

回答亮点：

重试不是简单“失败就再来一次”。必须结合错误码、超时预算、幂等性和限流，否则可能把小故障放大成大故障。

---

### 45. 什么是 deadline propagation？

deadline propagation 是指上游设置的超时截止时间能够传递到下游服务。

例如：

```text
Gateway 总超时 2s
  -> OrderService 收到请求时剩余 1.8s
  -> OrderService 调 UserService 时继续使用派生 context
  -> UserService 能感知 deadline
```

好处：

- 避免下游在上游已放弃后继续浪费资源。
- 防止调用链无限等待。
- 有助于整体超时预算管理。

Go 中只要正确使用 ctx 派生和传递，gRPC 会帮你传播 deadline。

---

## 十二、gRPC-Gateway 与对外 API

### 46. gRPC-Gateway 是什么？

gRPC-Gateway 是一个反向代理/转码层，可以把 HTTP/JSON 请求转换成 gRPC 请求。

常见架构：

```text
Browser / App / Third-party
  -> HTTP/JSON
  -> API Gateway
  -> gRPC internal services
```

优点：

- 内部服务使用 gRPC 强契约。
- 外部调用方使用熟悉的 HTTP/JSON。
- proto 可以同时生成 gRPC 代码和 HTTP Gateway 代码。

注意：

Gateway 不应该写核心业务逻辑。它主要负责协议转换、基础鉴权、日志、trace id 传递和错误格式统一。

---

### 47. gRPC-Gateway 如何定义 HTTP 路由？

在 proto 中使用 HTTP annotation：

```proto
import "google/api/annotations.proto";

service OrderService {
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse) {
    option (google.api.http) = {
      post: "/v1/orders"
      body: "*"
    };
  }

  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse) {
    option (google.api.http) = {
      get: "/v1/orders/{id}"
    };
  }
}
```

生成 Gateway 代码后，HTTP 请求会被转成对应 gRPC 调用。

常见问题：

- JSON 字段名和 proto 字段名不一致，例如 `user_id` 默认 JSON 是 `userId`。
- 忘记引入 `google/api/annotations.proto`。
- Gateway 生成代码没有配置。

---

### 48. Gateway 如何处理 gRPC 错误到 HTTP 状态码的映射？

常见映射：

| gRPC code | HTTP status |
| --- | --- |
| InvalidArgument | 400 |
| Unauthenticated | 401 |
| PermissionDenied | 403 |
| NotFound | 404 |
| AlreadyExists | 409 |
| FailedPrecondition | 400 或 412 |
| DeadlineExceeded | 504 |
| Unavailable | 503 |
| Internal | 500 |

生产中通常会统一错误响应格式：

```json
{
  "code": "NOT_FOUND",
  "message": "order not found",
  "traceId": "trace-001"
}
```

回答重点：

Gateway 层不应该把内部堆栈或敏感错误直接暴露给外部。要保留可定位信息，例如 trace id，同时隐藏内部实现细节。

---

## 十三、测试、压测与性能优化

### 49. gRPC 服务如何做单元测试？

如果业务逻辑和网络启动分离，可以直接实例化 service 测试方法。

示例：

```go
func TestGetUser(t *testing.T) {
	svc := user.NewService()

	resp, err := svc.GetUser(context.Background(), &userv1.GetUserRequest{Id: 1})
	if err != nil {
		t.Fatal(err)
	}
	if resp.GetUser().GetId() != 1 {
		t.Fatal("unexpected user id")
	}
}
```

适合测试：

- 参数校验。
- 错误码。
- 内存业务逻辑。
- 状态变化。

注意：

单元测试不一定要真的启动 TCP 端口。把业务 service 和 main 启动逻辑拆开，测试会更简单。

---

### 50. 什么是 bufconn？为什么适合 gRPC 测试？

`bufconn` 是 gRPC Go 提供的内存连接工具，可以在不监听真实 TCP 端口的情况下进行集成测试。

优点：

- 不占用端口。
- 测试速度快。
- 比直接调用 service 更接近真实 gRPC 调用。
- 能测试序列化、拦截器、错误码等链路。

典型结构：

```go
listener := bufconn.Listen(1024 * 1024)
server := grpc.NewServer()
userv1.RegisterUserServiceServer(server, user.NewService())

go server.Serve(listener)

ctx := context.Background()
conn, err := grpc.DialContext(ctx, "bufnet",
	grpc.WithContextDialer(func(context.Context, string) (net.Conn, error) {
		return listener.Dial()
	}),
	grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

---

### 51. 如何压测 gRPC 服务？

常用工具：

- ghz
- grpcurl 简单调试
- 自写 Go benchmark
- pprof
- Prometheus metrics

ghz 示例：

```powershell
ghz --insecure `
  --proto .\proto\user\v1\user.proto `
  --call user.v1.UserService.GetUser `
  -d '{\"id\":1}' `
  -c 20 `
  -n 1000 `
  localhost:50051
```

关注指标：

- QPS
- 平均延迟
- P95/P99
- 错误率
- status code distribution
- CPU、内存、goroutine、GC

不要只看 QPS。高 QPS 伴随高错误率没有意义。P95/P99 往往比平均延迟更能反映用户体验。

---

### 52. gRPC 性能优化可以从哪些方向入手？

方向：

- 连接复用，避免频繁 Dial。
- 合理设置超时，避免资源堆积。
- 控制 message 大小，避免大对象传输。
- 使用 streaming 处理大批量数据。
- 减少不必要的 metadata。
- 优化序列化对象分配。
- 服务端减少锁竞争。
- 下游数据库和缓存优化。
- 使用 pprof 分析 CPU、内存、goroutine。
- 使用 metrics 观察延迟分布和错误率。

回答注意：

不要一上来就说“调大并发”。性能优化应该先测量，再定位瓶颈，再验证优化效果。

---

### 53. gRPC 中大消息有什么问题？

大消息会带来：

- 序列化和反序列化耗时。
- 内存占用上升。
- GC 压力增加。
- 网络传输时间变长。
- 超过默认 message size 限制。

解决方式：

- 使用分页。
- 使用 streaming 分块传输。
- 避免在单个 RPC 中返回巨大列表。
- 对文件类数据使用对象存储，只在 gRPC 中传元数据。
- 必要时调整 max message size，但不应作为第一选择。

---

### 54. 如何排查 gRPC 调用慢？

排查顺序：

1. 看客户端是否设置了 timeout。
2. 看服务端日志中的 method、code、duration。
3. 看是否有下游调用慢。
4. 看 pprof：CPU、heap、goroutine、block、mutex。
5. 看 metrics：P95/P99、错误率、QPS。
6. 看 tracing：调用链中哪一段耗时最高。
7. 看网络和负载均衡：是否连接不均衡、丢包、跨机房。
8. 看数据库、缓存、外部依赖。

面试回答：

我不会只凭感觉优化，会先通过日志、metrics、tracing 和 pprof 定位慢在哪里，然后再针对性处理。

---

## 十四、微服务与项目实战

### 55. 在订单微服务项目中，服务应该如何拆分？

可以按职责拆分：

- UserService：用户创建、查询。
- ProductService：商品查询、库存扣减。
- OrderService：订单创建、查询、状态流。
- Gateway：对外 HTTP/JSON 入口。

边界原则：

- 每个服务拥有自己的数据和职责。
- OrderService 不直接读取 ProductService 的内部 map 或数据库。
- 跨服务调用通过 gRPC client。
- Gateway 不写核心业务逻辑。

面试亮点：

服务拆分不是为了数量多，而是为了边界清晰、职责明确、独立演进。

---

### 56. 创建订单链路如何设计？

简化链路：

```text
Gateway
  -> OrderService.CreateOrder
     -> UserService.GetUser
     -> ProductService.GetProduct
     -> ProductService.DeductStock
     -> 保存订单
     -> 返回订单
```

要考虑：

- 参数校验。
- 用户是否存在。
- 商品是否存在。
- 库存是否足够。
- 下游调用超时。
- 库存扣减和订单创建的一致性。
- 幂等 request_id。

回答时要主动提风险：

如果库存扣了但订单创建失败，会出现跨服务一致性问题。教学项目可以记录风险，真实项目需要 Saga、TCC、Outbox、事务消息或其他一致性方案。

---

### 57. gRPC 项目中如何处理跨服务一致性？

常见方案：

- 本地事务：只适合同一个数据库内。
- Saga：多个本地事务加补偿动作。
- TCC：Try、Confirm、Cancel，适合强业务控制。
- Outbox：本地事务写业务数据和事件表，再异步投递消息。
- 事务消息：依赖消息队列能力。
- 幂等和重试：保证重复请求不造成重复副作用。

回答建议：

gRPC 只解决服务通信，不自动解决分布式事务。跨服务一致性要在业务架构层设计，不能指望 RPC 框架帮我们保证。

---

### 58. 如何设计幂等？

幂等是指同一个请求重复执行多次，结果和执行一次一致。

常见做法：

- 客户端生成 `request_id`。
- 服务端用唯一索引或幂等表记录请求。
- 如果 request_id 已处理，直接返回之前结果。
- 对扣库存、支付、创建订单等写操作必须特别注意。

proto 示例：

```proto
message CreateOrderRequest {
  int64 user_id = 1;
  int64 product_id = 2;
  int32 quantity = 3;
  string request_id = 4;
}
```

面试亮点：

幂等不是只在客户端防抖，真正可靠的幂等要由服务端保证。

---

### 59. 如果下游服务不可用，OrderService 应该怎么办？

看场景：

- 用户服务不可用：创建订单不能继续，返回 `Unavailable` 或包装后的业务错误。
- 商品服务不可用：不能确认库存，不能创建订单。
- 通知服务不可用：如果通知不是主链路，可以异步重试或降级。

策略：

- 设置超时。
- 对幂等查询可以有限重试。
- 对写操作谨慎重试。
- 熔断和限流防止故障扩散。
- 返回明确错误码。
- 记录日志和 trace id。

回答重点：

不是所有下游失败都一样。要区分核心链路和非核心链路，区分可重试和不可重试。

---

### 60. 如何设计 gRPC 服务的日志？

建议日志字段：

- trace_id
- method
- peer
- code
- duration
- request_id
- user_id
- error message

示例：

```text
trace_id=abc method=/order.v1.OrderService/CreateOrder code=OK duration=35ms request_id=req-001
```

注意：

- 不要打印完整 token。
- 不要打印敏感个人信息。
- 大请求体不要全量打印。
- 错误日志要能关联 trace。

日志最好通过 interceptor 统一记录，而不是每个 handler 手写。

---

### 61. 如何设计 gRPC 服务的监控指标？

常见指标：

- 请求总数：按 method、code 维度。
- 请求延迟：P50/P95/P99。
- 当前连接数。
- stream 数量。
- 错误率。
- goroutine 数量。
- CPU、内存、GC。
- 下游调用耗时。

Prometheus 指标示例：

```text
grpc_server_handled_total{grpc_method="CreateOrder",grpc_code="OK"}
grpc_server_handling_seconds_bucket{grpc_method="CreateOrder"}
```

面试表达：

日志用于排查单次请求，metrics 用于看整体趋势，tracing 用于看跨服务链路。三者要结合。

---

## 十五、常见场景题

### 62. gRPC 调用偶尔 DeadlineExceeded，你怎么排查？

排查步骤：

1. 确认客户端 timeout 是否过短。
2. 查看服务端是否真的收到请求。
3. 查看服务端处理耗时分布，尤其 P95/P99。
4. 查看下游依赖耗时。
5. 查看是否有 GC、锁竞争、goroutine 堆积。
6. 查看网络是否抖动。
7. 查看负载是否不均衡。
8. 用 tracing 定位慢在哪一段。

可能原因：

- 下游数据库慢。
- 服务端 goroutine 阻塞。
- 客户端连接复用不当。
- 某些实例负载过高。
- timeout 预算不合理。

回答亮点：

偶发超时通常要看尾延迟，而不是只看平均延迟。

---

### 63. gRPC 服务返回 Unavailable，可能是什么原因？

可能原因：

- 服务端未启动。
- 端口不可达。
- DNS 或服务发现异常。
- 负载均衡没有可用实例。
- TLS 握手失败。
- 服务端连接被关闭。
- 服务端过载。
- Kubernetes readiness 失败导致 endpoints 为空。

排查：

- grpcurl 直连实例。
- 检查服务端日志。
- 检查健康检查。
- 检查客户端 resolver 和 balancer。
- 检查网络、防火墙、证书。

---

### 64. gRPC 接口上线后如何保证兼容？

措施：

- proto 字段只新增，不随意删除或改语义。
- 删除字段使用 reserved。
- 包名带版本，例如 `order.v1`。
- 使用 buf breaking 检查。
- 老客户端未传新字段时，服务端要有默认处理。
- 不兼容变更新开版本，例如 `order.v2`。
- 灰度发布，观察错误率。

回答示例：

我会把 proto 当成服务契约管理，CI 中加入 lint 和 breaking check，避免误改字段编号或删除已发布字段。

---

### 65. 如何让 gRPC 服务优雅关闭？

Go 服务端可以监听系统信号，收到退出信号后调用 `GracefulStop()`。

示例：

```go
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

<-quit
grpcServer.GracefulStop()
```

`GracefulStop` 会停止接收新连接，并等待已有 RPC 完成。

如果等待太久，可以配合超时后调用 `Stop()` 强制关闭。

生产注意：

- Kubernetes 中要配合 readiness 先摘流量。
- 设置 terminationGracePeriodSeconds。
- 长流式连接需要有最大生命周期或服务端退出通知。

---

### 66. 如何处理 gRPC 中的限流？

限流可以放在：

- Gateway 层。
- 服务端 interceptor。
- 服务网格或 Envoy。
- 下游依赖保护层。

限流维度：

- 按 method。
- 按 user_id。
- 按 tenant_id。
- 按 client app。
- 按 IP。

超过限制时可以返回：

```go
status.Error(codes.ResourceExhausted, "rate limit exceeded")
```

注意：

限流要配合监控和告警。只有限流没有降级策略，用户体验可能仍然很差。

---

### 67. 如何处理 gRPC 消息压缩？

gRPC 支持压缩，例如 gzip。

适合：

- 响应体较大。
- 网络带宽紧张。

不适合：

- 小消息频繁调用，压缩开销可能大于收益。
- CPU 已经是瓶颈。

面试表达：

压缩是带宽和 CPU 的权衡，要压测验证。不能默认所有接口都开压缩。

---

### 68. gRPC 和消息队列有什么区别？

gRPC 是同步或流式 RPC 调用，强调请求响应。

消息队列是异步通信，强调解耦、削峰、重试、最终一致性。

对比：

| 场景 | 更适合 |
| --- | --- |
| 查询用户信息 | gRPC |
| 创建订单主链路同步校验 | gRPC |
| 订单创建后发通知 | 消息队列 |
| 事件广播 | 消息队列 |
| 实时双向通信 | gRPC Streaming 或 WebSocket |

回答：

两者不是替代关系。微服务中常见组合是：核心同步查询用 gRPC，非核心副作用和事件驱动用消息队列。

---

### 69. 什么时候不适合使用 gRPC？

不适合或需要谨慎：

- 主要面向浏览器直接调用。
- 对外开放 API，需要广泛第三方接入。
- 团队缺少 proto 和工具链治理经验。
- 调试和可观测性建设不足。
- 简单 CRUD 后台，HTTP JSON 已经足够。

回答注意：

不要把 gRPC 神化。技术选型要看调用方、团队能力、调试成本、生态和性能需求。

---

### 70. 你在 gRPC 项目中最容易踩的坑是什么？

可以回答几个真实感强的点：

- 每次请求都新建连接，导致延迟和资源消耗高。
- 没有设置 timeout，故障时 goroutine 堆积。
- proto 字段编号随意修改，导致兼容性问题。
- 所有错误都返回 Internal，调用方无法处理。
- 只写 Unary interceptor，忘记 Stream interceptor。
- Gateway 层混入业务逻辑，边界变乱。
- 流式 RPC 没处理取消，导致 goroutine 泄漏。
- 压测只看 QPS，不看错误率和 P99。

好的回答方式：

```text
我之前会特别注意连接复用、超时和错误码，因为这几个问题在 demo 里不明显，但到微服务项目中会直接影响稳定性和排查效率。
```

---

## 十六、项目追问题：订单微服务系统

### 71. 如果面试官问：你的订单项目为什么要拆 UserService、ProductService、OrderService？

参考答案：

因为它们的业务职责不同，变化原因也不同。UserService 管用户资料，ProductService 管商品和库存，OrderService 管订单生命周期。拆分后，OrderService 通过 gRPC client 调用用户和商品服务，而不是直接访问它们的数据结构。

这样做的好处是：

- 服务边界清晰。
- 每个服务可以独立测试。
- proto 契约明确。
- 后续可以独立替换存储或扩容。

但我也会说明：学习项目拆分是为了练习 gRPC 和服务边界，真实项目是否拆这么细，要看团队规模、业务复杂度和运维能力。

---

### 72. 如果库存扣减成功，但订单创建失败怎么办？

参考答案：

这是典型跨服务一致性问题。gRPC 只负责服务通信，不保证分布式事务。

解决思路：

- 学习项目：记录风险，返回错误，人工或任务补偿。
- 生产项目：可以用 Saga，扣库存后如果订单创建失败，调用补偿接口释放库存。
- 也可以使用 Outbox 模式或事务消息，把订单事件可靠投递出去。
- 所有写操作要设计幂等键，避免重试造成重复扣减。

面试时要主动承认这个风险，不要说“调用成功就没问题”。

---

### 73. 如果 Gateway 收到 HTTP 请求，如何把 trace id 传给内部 gRPC？

参考答案：

Gateway 从 HTTP header 中读取 `X-Trace-Id`，然后放入 gRPC metadata。

示例：

```go
runtime.WithMetadata(func(ctx context.Context, r *http.Request) metadata.MD {
	traceID := r.Header.Get("X-Trace-Id")
	if traceID == "" {
		return nil
	}
	return metadata.Pairs("x-trace-id", traceID)
})
```

内部服务的 interceptor 再从 metadata 中读取 trace id，写入日志和 tracing span。

---

### 74. 如何证明你的 gRPC 项目是可测试的？

参考答案：

我会从三个层次说明：

1. 单元测试：直接实例化 service，测试参数校验和业务逻辑。
2. bufconn 集成测试：不占端口，但走完整 gRPC 调用链，测试拦截器、错误码和序列化。
3. 端到端测试：启动多个服务，用 grpcurl 或自动化脚本验证创建订单链路。

另外，我会让 main 函数只做装配，业务逻辑放在 internal app 包里，这样测试不依赖真实端口。

---

### 75. 如何向面试官解释你对 gRPC 生产化的理解？

参考答案：

我理解的 gRPC 生产化不只是能跑通 RPC，而是要具备：

- 契约治理：proto lint、breaking check、版本管理。
- 稳定性：timeout、retry、限流、熔断、健康检查。
- 安全：TLS/mTLS、token、权限控制。
- 可观测性：日志、metrics、tracing。
- 可测试性：单元测试、bufconn、集成测试、压测。
- 运维能力：优雅关闭、服务发现、负载均衡、Kubernetes readiness。
- 故障处理：明确错误码、降级策略、告警和排查手段。

一句话总结：

```text
能跑通 gRPC demo 只是入门，能把它放进真实微服务体系并可观测、可治理、可演进，才是生产化。
```

---

## 十七、面试快速背诵版

### 高频关键词

```text
HTTP/2
Protobuf
强契约
字段编号
go_package
Unary
Streaming
context
deadline
status code
metadata
interceptor
TLS/mTLS
health check
reflection
service discovery
load balancing
retry
idempotency
gRPC-Gateway
bufconn
ghz
P95/P99
tracing
metrics
pprof
graceful shutdown
```

### 高频一句话答案

- gRPC 是基于 HTTP/2 和 Protobuf 的高性能 RPC 框架。
- Protobuf 字段编号是二进制兼容的关键，发布后不能随意修改。
- gRPC 更适合内部服务强契约通信，HTTP/JSON 更适合对外开放。
- ClientConn 可以复用，不应该每次请求都 Dial。
- 每个 RPC 都应该有 timeout，避免故障扩散。
- 错误码要有语义，不能所有错误都返回 Internal。
- Metadata 用来传 token、trace id 等调用上下文。
- Interceptor 用来统一处理日志、鉴权、恢复、metrics、tracing。
- Stream RPC 要特别注意取消、背压和 goroutine 生命周期。
- 重试必须结合幂等性，否则可能造成重复扣库存或重复下单。
- Gateway 负责协议转换，不应该写核心业务。
- 压测不能只看 QPS，还要看错误率和 P95/P99。

---

## 十八、面试自测清单

面试前可以用下面的清单自测。如果能不看答案讲清楚，gRPC 面试基本就比较稳。

```text
[ ] 能解释 gRPC、HTTP/2、Protobuf 的关系
[ ] 能写出一个简单 proto
[ ] 能解释字段编号和兼容性
[ ] 能说清 package 和 go_package
[ ] 能写 Go 服务端注册流程
[ ] 能写 Go 客户端调用流程
[ ] 能解释四种 RPC 模式
[ ] 能说明 streaming 的取消处理
[ ] 能解释 context deadline
[ ] 能区分常见 status code
[ ] 能使用 metadata 传 token 和 trace id
[ ] 能写 unary interceptor
[ ] 能解释 stream interceptor 的特殊性
[ ] 能说明 TLS 和 mTLS
[ ] 能解释 health check 和 reflection
[ ] 能说明服务发现和负载均衡
[ ] 能解释 retry 和幂等
[ ] 能说明 gRPC-Gateway 的作用
[ ] 能用 bufconn 做测试
[ ] 能用 ghz 做压测
[ ] 能从日志、metrics、tracing、pprof 排查慢调用
[ ] 能讲清一个 gRPC 微服务项目的边界和风险
```

---

## 十九、最后的面试表达建议

面试官问 gRPC 时，通常不是只考 API 记忆，而是想判断你是否具备后端工程化思维。

回答时尽量做到：

- 不只说概念，还能说代码怎么写。
- 不只说优点，也能说限制和坑。
- 不只说 demo，还能说生产环境怎么治理。
- 不只说调用成功，还能说失败、超时、重试、幂等、观测。

一个比较成熟的回答习惯是：

```text
这个问题在 demo 里可以这样做；
但如果放到生产环境，我还会考虑超时、错误码、可观测性和兼容性。
```

这句话背后体现的是工程判断力。Go 后端面试里，工程判断力往往比单个 API 更重要。
