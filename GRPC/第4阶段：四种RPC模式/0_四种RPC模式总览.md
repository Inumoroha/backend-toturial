# 0. 四种 RPC 模式总览

本节目标：理解 Unary、Server Streaming、Client Streaming、Bidirectional Streaming 的区别和使用场景。

gRPC 不只有一次请求一次响应。基于 HTTP/2，它天然支持流式传输。学会四种 RPC 模式后，你可以处理查询、导出、上传、实时通信等不同场景。

---

## 一、核心直觉

- Unary：一次请求，一次响应，最常见。
- Server Streaming：一次请求，多次响应，适合导出、进度推送。
- Client Streaming：多次请求，一次响应，适合上传、批量提交。
- Bidirectional Streaming：双方都可以连续发送，适合聊天、实时协作。
- Streaming 的核心是循环 `Send` 和 `Recv`，并正确处理 `io.EOF`。

---

## 二、动手步骤

1. 先在 proto 中定义四种方法。
2. 分别实现服务端逻辑。
3. 分别实现客户端调用。
4. 观察每种模式的阻塞点。
5. 加入 context 取消测试。

---

## 三、参考代码或命令

```proto
service OrderService {
  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
  rpc ListOrders(ListOrdersRequest) returns (stream Order);
  rpc CreateOrders(stream CreateOrderRequest) returns (CreateOrdersResponse);
  rpc ChatOrderStatus(stream OrderStatusMessage) returns (stream OrderStatusMessage);
}
```

---

## 四、常见问题

- 不知道什么时候结束流：Client Streaming 要 `CloseAndRecv`，Server Streaming 要读到 `io.EOF`。
- 在流里忽略错误：任何一次 Send/Recv 都可能失败。
- 把大列表都塞进一个响应：如果数据很多，Server Streaming 更合适。

---

## 五、练习任务

- 为 `OrderService` 定义四种 RPC 方法。
- 每种方法写一个最小 demo。
- 记录每种模式适合的业务场景。

---

## 六、完成标准

- 能从业务场景选择 RPC 模式。
- 能解释 stream 在请求侧还是响应侧。
- 能处理基本的 Send/Recv/EOF。


---

## 七、本阶段实验工程结构

继续使用前面阶段的 `grpc-learning` 工程。为了避免和 `UserService` 混在一起，本阶段新增一个订单服务：

```text
grpc-learning/
  proto/
    order/
      v1/
        order.proto
        order.pb.go
        order_grpc.pb.go
  cmd/
    order-server/
      main.go
    order-client/
      main.go
```

如果你还没有创建目录，可以执行：

```powershell
New-Item -ItemType Directory -Force proto/order/v1
New-Item -ItemType Directory -Force cmd/order-server
New-Item -ItemType Directory -Force cmd/order-client
```

本阶段会在一个 `OrderService` 中同时定义四种 RPC 模式，这样你可以直接比较它们的差异。

---

## 八、完整 proto 设计

创建 `proto/order/v1/order.proto`：

```proto
syntax = "proto3";

package order.v1;

option go_package = "example.com/grpc-learning/proto/order/v1;orderv1";

message Order {
  int64 id = 1;
  int64 user_id = 2;
  string product_name = 3;
  int32 quantity = 4;
  string status = 5;
}

message GetOrderRequest {
  int64 id = 1;
}

message GetOrderResponse {
  Order order = 1;
}

message ListOrdersRequest {
  int64 user_id = 1;
}

message CreateOrderRequest {
  int64 user_id = 1;
  string product_name = 2;
  int32 quantity = 3;
}

message CreateOrdersResponse {
  int64 created_count = 1;
}

message OrderStatusMessage {
  int64 order_id = 1;
  string text = 2;
}

service OrderService {
  rpc GetOrder(GetOrderRequest) returns (GetOrderResponse);
  rpc ListOrders(ListOrdersRequest) returns (stream Order);
  rpc CreateOrders(stream CreateOrderRequest) returns (CreateOrdersResponse);
  rpc ChatOrderStatus(stream OrderStatusMessage) returns (stream OrderStatusMessage);
}
```

四个方法分别对应：

```text
GetOrder          Unary RPC
ListOrders        Server Streaming
CreateOrders      Client Streaming
ChatOrderStatus   Bidirectional Streaming
```

---

## 九、生成代码

在工程根目录执行：

```powershell
protoc --go_out=. --go_opt=paths=source_relative `
  --go-grpc_out=. --go-grpc_opt=paths=source_relative `
  proto/order/v1/order.proto
```

预期生成：

```text
proto/order/v1/order.pb.go
proto/order/v1/order_grpc.pb.go
```

检查：

```powershell
Get-ChildItem proto/order/v1
```

你应该看到：

```text
order.proto
order.pb.go
order_grpc.pb.go
```

---

## 十、四种 RPC 的方法签名对比

生成后打开 `order_grpc.pb.go`，你会看到不同方法对应不同 Go 签名。

Unary：

```go
GetOrder(context.Context, *GetOrderRequest) (*GetOrderResponse, error)
```

Server Streaming：

```go
ListOrders(*ListOrdersRequest, OrderService_ListOrdersServer) error
```

Client Streaming：

```go
CreateOrders(OrderService_CreateOrdersServer) error
```

Bidirectional Streaming：

```go
ChatOrderStatus(OrderService_ChatOrderStatusServer) error
```

这个差异非常关键。Unary 方法直接返回 response；streaming 方法通常拿到 stream 对象，通过 `Send`、`Recv` 完成消息传输。

---

## 十一、本阶段完整操作步骤

按下面顺序学习：

1. 创建 `order.proto`。
2. 生成 `order.pb.go` 和 `order_grpc.pb.go`。
3. 先实现 `GetOrder`，确认普通 Unary 仍然能跑。
4. 实现 `ListOrders`，学习服务端连续 `Send`。
5. 实现 `CreateOrders`，学习服务端循环 `Recv` 和 `SendAndClose`。
6. 实现 `ChatOrderStatus`，学习双向流的发送和接收。
7. 为每种模式写一个 client 调用函数。
8. 用正常输入、提前取消、空数据三类情况测试。

---

## 十二、运行命令

启动服务端：

```powershell
go run ./cmd/order-server
```

运行客户端：

```powershell
go run ./cmd/order-client
```

如果你在客户端里用命令行参数区分模式，可以设计成：

```powershell
go run ./cmd/order-client unary
go run ./cmd/order-client server-stream
go run ./cmd/order-client client-stream
go run ./cmd/order-client bidi-stream
```

学习阶段也可以先在 `main.go` 里依次调用四个函数。

---

## 十三、预期输出

完整跑通后，客户端应该能看到类似：

```text
=== Unary ===
order id=1 product=Book quantity=2 status=CREATED

=== Server Streaming ===
order id=1 product=Book
order id=2 product=Keyboard
order id=3 product=Mouse
server stream finished

=== Client Streaming ===
created count=3

=== Bidirectional Streaming ===
recv: ack: client says order 1 paid
recv: ack: client says order 1 shipped
```

输出不需要完全一样，但必须能清楚看出四种模式的差别。

---

## 十四、常见错误排查

### 1. streaming 方法签名写错

不要自己猜签名。先打开 `order_grpc.pb.go`，复制生成接口中的方法签名。

### 2. 客户端忘记处理 io.EOF

Server Streaming 正常结束时，客户端 `Recv()` 会返回 `io.EOF`。这不是错误，是结束信号。

### 3. Client Streaming 忘记 CloseAndRecv

客户端发送完所有消息后必须调用：

```go
resp, err := stream.CloseAndRecv()
```

否则服务端可能一直等待更多请求。

### 4. 双向流读写顺序导致卡住

双向流如果双方都在等对方先发，容易死锁。学习阶段可以先写成“客户端 Send 一条，服务端回复一条，客户端 Recv 一条”。

---

## 十五、练习任务

1. 在 `OrderService` 中定义四种 RPC 方法。
2. 为每种 RPC 写一个最小 server 实现。
3. 为每种 RPC 写一个 client 调用函数。
4. 给 Server Streaming 增加 500ms 间隔，观察客户端逐条输出。
5. 给 Client Streaming 发送 5 个订单，确认汇总数量。
6. 给双向流发送 3 条消息，确认收到 3 条 ack。

---

## 十六、完成标准

完成本阶段总览后，你应该能：

```text
写出包含四种 RPC 模式的 proto
生成 Go 代码
说清四种模式的 Go 方法签名差异
知道 Send、Recv、SendAndClose、CloseAndRecv 分别在哪里用
知道 io.EOF 是 stream 正常结束信号
知道双向流为什么容易卡住
```

---

## 教程闭环检查

为了保证本节不是只停留在概念层面，学习时请按下面闭环完成：

1. **完整操作步骤**：创建 `order.proto`，生成代码，并准备 order server/client 目录。
2. **完整代码**：使用正文中的完整 proto，后续小节再补完整 server/client 代码。
3. **运行命令**：至少执行一次 `protoc` 生成命令，并确认生成文件存在。
4. **预期输出**：对照生成文件列表和后续四种 RPC 的运行输出。
5. **常见错误排查**：遇到签名、EOF、CloseAndRecv、双向流阻塞问题时回到本节排查。
6. **练习任务**：完成本节列出的六个练习。
7. **完成标准**：能不看资料说清四种 RPC 模式差异。