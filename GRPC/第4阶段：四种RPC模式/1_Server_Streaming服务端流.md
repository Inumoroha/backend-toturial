# 1. Server Streaming 服务端流

本节目标：掌握服务端连续返回多条消息的写法。

Server Streaming 是“一次请求，多次响应”。客户端发起一次请求，服务端可以持续发送多条消息，直到发送完成或发生错误。

---

## 一、核心直觉

- proto 中响应类型前加 `stream`。
- 服务端方法没有普通 response 返回值，而是拿到 stream 对象。
- 服务端使用 `stream.Send` 逐条发送。
- 客户端使用 `Recv` 循环读取，直到 `io.EOF`。

---

## 二、动手步骤

1. 定义 `ListOrders`。
2. 服务端模拟订单列表。
3. 每隔一小段时间发送一条订单。
4. 客户端循环接收并打印。
5. 加入 context 超时观察中断。

---

## 三、参考代码或命令

```proto
rpc ListOrders(ListOrdersRequest) returns (stream Order);
```

服务端：

```go
func (s *orderServer) ListOrders(req *orderv1.ListOrdersRequest, stream orderv1.OrderService_ListOrdersServer) error {
    for _, order := range orders {
        if err := stream.Send(order); err != nil {
            return err
        }
    }
    return nil
}
```

客户端：

```go
stream, err := client.ListOrders(ctx, req)
for {
    order, err := stream.Recv()
    if errors.Is(err, io.EOF) {
        break
    }
    if err != nil {
        return err
    }
    log.Println(order)
}
```

---

## 四、常见问题

- 服务端一次性查出所有数据再发送：数据大时仍然占内存，真实项目要分页或游标。
- 客户端不处理 `io.EOF`：正常结束会被误当成错误。
- 发送时不关注 context：客户端取消后服务端还在干活。

---

## 五、练习任务

- 实现一个订单列表流，每次返回一条订单。
- 模拟 100 条数据，观察客户端输出。
- client 设置 1 秒超时，观察服务端 Send 错误。

---

## 六、完成标准

- 服务端能连续发送消息。
- 客户端能正确读到 EOF。
- 能说明 Server Streaming 适合哪些场景。


---

## 七、完整操作步骤

Server Streaming 的目标是：客户端发起一次请求，服务端连续返回多条订单。

本节要完成：

1. 确认 `order.proto` 中有 `ListOrders`。
2. 在 order server 中实现 `ListOrders`。
3. 在 order client 中实现 `listOrders` 调用函数。
4. 运行 server。
5. 运行 client，观察订单逐条输出。
6. 测试客户端超时取消。

proto 方法：

```proto
rpc ListOrders(ListOrdersRequest) returns (stream Order);
```

生成的服务端签名类似：

```go
ListOrders(*orderv1.ListOrdersRequest, orderv1.OrderService_ListOrdersServer) error
```

注意：这个方法没有直接返回 response，而是通过 `stream.Send` 多次发送。

---

## 八、完整服务端代码

在 `cmd/order-server/main.go` 中准备一个 server：

```go
type orderServer struct {
    orderv1.UnimplementedOrderServiceServer
    orders []*orderv1.Order
}

func newOrderServer() *orderServer {
    return &orderServer{
        orders: []*orderv1.Order{
            {Id: 1, UserId: 100, ProductName: "Book", Quantity: 2, Status: "CREATED"},
            {Id: 2, UserId: 100, ProductName: "Keyboard", Quantity: 1, Status: "PAID"},
            {Id: 3, UserId: 100, ProductName: "Mouse", Quantity: 1, Status: "SHIPPED"},
            {Id: 4, UserId: 200, ProductName: "Monitor", Quantity: 1, Status: "CREATED"},
        },
    }
}
```

实现 `ListOrders`：

```go
func (s *orderServer) ListOrders(req *orderv1.ListOrdersRequest, stream orderv1.OrderService_ListOrdersServer) error {
    if req.UserId <= 0 {
        return status.Error(codes.InvalidArgument, "user_id must be positive")
    }

    for _, order := range s.orders {
        if order.UserId != req.UserId {
            continue
        }

        select {
        case <-stream.Context().Done():
            log.Printf("client canceled ListOrders: %v", stream.Context().Err())
            return stream.Context().Err()
        case <-time.After(500 * time.Millisecond):
            if err := stream.Send(order); err != nil {
                return err
            }
        }
    }

    return nil
}
```

需要 import：

```go
import (
    "log"
    "time"

    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)
```

`time.After` 只是为了让你看到“逐条返回”的效果。真实项目中可能是边查数据库边发送。

---

## 九、服务端 main 函数

如果你还没有 order server，可以写一个完整入口：

```go
func main() {
    lis, err := net.Listen("tcp", ":50052")
    if err != nil {
        log.Fatalf("listen: %v", err)
    }

    server := grpc.NewServer()
    orderv1.RegisterOrderServiceServer(server, newOrderServer())
    reflection.Register(server)

    log.Println("order gRPC server listening on :50052")
    if err := server.Serve(lis); err != nil {
        log.Fatalf("serve: %v", err)
    }
}
```

本阶段使用 `:50052`，避免和前面 UserService 的 `:50051` 冲突。

---

## 十、完整客户端代码

在 `cmd/order-client/main.go` 中写调用函数：

```go
func listOrders(client orderv1.OrderServiceClient) error {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    stream, err := client.ListOrders(ctx, &orderv1.ListOrdersRequest{UserId: 100})
    if err != nil {
        return err
    }

    for {
        order, err := stream.Recv()
        if errors.Is(err, io.EOF) {
            log.Println("server stream finished")
            return nil
        }
        if err != nil {
            return err
        }

        log.Printf("order id=%d product=%s quantity=%d status=%s",
            order.GetId(), order.GetProductName(), order.GetQuantity(), order.GetStatus())
    }
}
```

完整 client main 可以这样写：

```go
func main() {
    conn, err := grpc.Dial("localhost:50052", grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        log.Fatalf("dial: %v", err)
    }
    defer conn.Close()

    client := orderv1.NewOrderServiceClient(conn)

    if err := listOrders(client); err != nil {
        st, _ := status.FromError(err)
        log.Fatalf("ListOrders failed: code=%s message=%s", st.Code(), st.Message())
    }
}
```

---

## 十一、运行命令

终端 1：

```powershell
go run ./cmd/order-server
```

预期输出：

```text
order gRPC server listening on :50052
```

终端 2：

```powershell
go run ./cmd/order-client
```

预期输出：

```text
order id=1 product=Book quantity=2 status=CREATED
order id=2 product=Keyboard quantity=1 status=PAID
order id=3 product=Mouse quantity=1 status=SHIPPED
server stream finished
```

你会看到三条订单不是一次性全部出现，而是大约每 500ms 出现一条。

---

## 十二、测试取消

把客户端 timeout 改短：

```go
ctx, cancel := context.WithTimeout(context.Background(), 700*time.Millisecond)
```

再次运行 client，可能只收到一条订单，然后报错：

```text
ListOrders failed: code=DeadlineExceeded message=context deadline exceeded
```

服务端日志可能看到：

```text
client canceled ListOrders: context deadline exceeded
```

这说明客户端取消会传播到服务端。

---

## 十三、常见错误排查

### 1. 把 io.EOF 当错误

Server Streaming 正常结束时，`Recv()` 返回 `io.EOF`。正确处理：

```go
if errors.Is(err, io.EOF) {
    return nil
}
```

不要直接 `return err`，否则正常结束会被打印成错误。

### 2. 服务端没有 return nil

服务端发送完所有数据后应该 `return nil`，表示流正常结束。

### 3. 一次性把所有数据塞到 response

如果数据很多，不要设计成：

```proto
message ListOrdersResponse {
  repeated Order orders = 1;
}
```

然后一次返回几十万条。Server Streaming 更适合边生成边发送。

### 4. 忘记检查 context

如果客户端已经取消，服务端还在继续查数据和发送，会浪费资源。

---

## 十四、练习任务

1. 把 `UserId` 改成 200，确认只返回一条订单。
2. 把 `UserId` 改成 999，确认没有订单但正常 EOF。
3. 把 timeout 改成 700ms，观察取消。
4. 给每条订单发送前打印服务端日志。
5. 增加 10 条订单，观察流式输出效果。

---

## 十五、完成标准

你完成本节后，应该能：

```text
写出 Server Streaming proto
实现服务端 stream.Send
实现客户端循环 stream.Recv
正确处理 io.EOF
正确处理客户端取消
说清 Server Streaming 适合导出、进度推送、列表逐条返回等场景
```

---

## 教程闭环检查

1. **完整操作步骤**：按本节步骤补齐 `ListOrders` server 和 client。
2. **完整代码**：使用正文中的 server、client、main 函数代码拼出可运行 demo。
3. **运行命令**：分别运行 `go run ./cmd/order-server` 和 `go run ./cmd/order-client`。
4. **预期输出**：客户端应逐条打印订单，最后打印 `server stream finished`。
5. **常见错误排查**：重点检查 `io.EOF`、context 取消、服务端是否 `return nil`。
6. **练习任务**：完成不同 user_id、短 timeout、多订单输出练习。
7. **完成标准**：能独立解释 Server Streaming 的 Send/Recv/EOF 流程。