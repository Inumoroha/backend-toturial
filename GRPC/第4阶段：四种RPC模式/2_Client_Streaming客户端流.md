# 2. Client Streaming 客户端流

本节目标：掌握客户端连续发送多条消息，服务端最终返回汇总结果的写法。

Client Streaming 是“多次请求，一次响应”。典型场景是上传文件、批量导入、客户端采集数据后汇总提交。

---

## 一、核心直觉

- proto 中请求类型前加 `stream`。
- 服务端循环 `Recv()`，直到收到 `io.EOF`。
- 客户端循环 `Send()`，发送完后调用 `CloseAndRecv()`。
- 服务端最终返回一个汇总响应。

---

## 二、动手步骤

1. 定义 `CreateOrders`。
2. 客户端连续发送多个订单创建请求。
3. 服务端统计数量和失败项。
4. 客户端关闭发送并等待响应。

---

## 三、参考代码或命令

```proto
rpc CreateOrders(stream CreateOrderRequest) returns (CreateOrdersResponse);
```

服务端：

```go
func (s *orderServer) CreateOrders(stream orderv1.OrderService_CreateOrdersServer) error {
    var count int64
    for {
        req, err := stream.Recv()
        if errors.Is(err, io.EOF) {
            return stream.SendAndClose(&orderv1.CreateOrdersResponse{CreatedCount: count})
        }
        if err != nil {
            return err
        }
        _ = req
        count++
    }
}
```

客户端：

```go
stream, err := client.CreateOrders(ctx)
for _, req := range requests {
    if err := stream.Send(req); err != nil {
        return err
    }
}
resp, err := stream.CloseAndRecv()
```

---

## 四、常见问题

- 客户端忘记 `CloseAndRecv`：服务端不知道客户端已经发送完成。
- 服务端不处理 `io.EOF`：正常结束无法返回汇总结果。
- 一边发送一边不考虑限速：大量数据可能造成内存和网络压力。

---

## 五、练习任务

- 模拟批量创建 10 个订单。
- 服务端统计成功数量。
- 加入一个非法订单，思考是立刻失败还是汇总失败项。

---

## 六、完成标准

- 能完成 Client Streaming 的收发闭环。
- 能正确处理 EOF。
- 能说清楚批量场景下的错误策略。


---

## 七、完整操作步骤

Client Streaming 的目标是：客户端连续发送多个订单创建请求，服务端全部接收后返回一个汇总结果。

本节要完成：

1. 确认 proto 中有 `CreateOrders`。
2. 服务端实现循环 `Recv()`。
3. 服务端收到 `io.EOF` 后调用 `SendAndClose()`。
4. 客户端循环 `Send()` 多个请求。
5. 客户端调用 `CloseAndRecv()` 获取汇总响应。
6. 测试正常发送、空流、非法请求三种情况。

proto 方法：

```proto
rpc CreateOrders(stream CreateOrderRequest) returns (CreateOrdersResponse);
```

生成的服务端签名类似：

```go
CreateOrders(orderv1.OrderService_CreateOrdersServer) error
```

---

## 八、完整服务端代码

在 `orderServer` 中实现：

```go
func (s *orderServer) CreateOrders(stream orderv1.OrderService_CreateOrdersServer) error {
    var createdCount int64

    for {
        req, err := stream.Recv()
        if errors.Is(err, io.EOF) {
            return stream.SendAndClose(&orderv1.CreateOrdersResponse{
                CreatedCount: createdCount,
            })
        }
        if err != nil {
            return err
        }

        if req.UserId <= 0 {
            return status.Error(codes.InvalidArgument, "user_id must be positive")
        }
        if req.ProductName == "" {
            return status.Error(codes.InvalidArgument, "product_name is required")
        }
        if req.Quantity <= 0 {
            return status.Error(codes.InvalidArgument, "quantity must be positive")
        }

        createdCount++
        log.Printf("received order: user_id=%d product=%s quantity=%d",
            req.GetUserId(), req.GetProductName(), req.GetQuantity())
    }
}
```

需要 import：

```go
import (
    "errors"
    "io"
    "log"

    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)
```

这里没有真的保存订单，只统计数量。项目实战阶段会把订单保存到内存或数据库。

---

## 九、完整客户端代码

```go
func createOrders(client orderv1.OrderServiceClient) error {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    stream, err := client.CreateOrders(ctx)
    if err != nil {
        return err
    }

    requests := []*orderv1.CreateOrderRequest{
        {UserId: 100, ProductName: "Book", Quantity: 2},
        {UserId: 100, ProductName: "Keyboard", Quantity: 1},
        {UserId: 100, ProductName: "Mouse", Quantity: 1},
    }

    for _, req := range requests {
        log.Printf("send order: product=%s quantity=%d", req.GetProductName(), req.GetQuantity())
        if err := stream.Send(req); err != nil {
            return err
        }
        time.Sleep(300 * time.Millisecond)
    }

    resp, err := stream.CloseAndRecv()
    if err != nil {
        return err
    }

    log.Printf("created count=%d", resp.GetCreatedCount())
    return nil
}
```

在 main 中调用：

```go
if err := createOrders(client); err != nil {
    st, _ := status.FromError(err)
    log.Fatalf("CreateOrders failed: code=%s message=%s", st.Code(), st.Message())
}
```

---

## 十、运行命令

终端 1：

```powershell
go run ./cmd/order-server
```

终端 2：

```powershell
go run ./cmd/order-client
```

预期客户端输出：

```text
send order: product=Book quantity=2
send order: product=Keyboard quantity=1
send order: product=Mouse quantity=1
created count=3
```

预期服务端输出：

```text
received order: user_id=100 product=Book quantity=2
received order: user_id=100 product=Keyboard quantity=1
received order: user_id=100 product=Mouse quantity=1
```

---

## 十一、空流测试

如果客户端不发送任何请求，直接：

```go
resp, err := stream.CloseAndRecv()
```

服务端会在第一次 `Recv()` 时收到 `io.EOF`，然后返回：

```text
created count=0
```

这说明 Client Streaming 可以表达“客户端发送了 0 到多条消息”。

---

## 十二、非法请求测试

把其中一个请求改成：

```go
{UserId: 100, ProductName: "", Quantity: 1}
```

运行后可能看到：

```text
CreateOrders failed: code=InvalidArgument message=product_name is required
```

一旦服务端返回错误，整个流就失败了。已经发送过的请求是否保存，取决于你的业务实现。真实批量导入场景要设计清楚：

```text
遇到错误立即失败
还是记录失败项，最后返回成功和失败明细
```

---

## 十三、常见错误排查

### 1. 客户端忘记 CloseAndRecv

如果只 Send 不 CloseAndRecv，服务端不知道客户端已经发送完成，可能一直等。

正确写法：

```go
resp, err := stream.CloseAndRecv()
```

### 2. 服务端不处理 io.EOF

服务端收到 `io.EOF` 表示客户端发送完了。此时应该返回汇总响应：

```go
return stream.SendAndClose(resp)
```

### 3. Send 返回 EOF

如果服务端已经因为错误关闭流，客户端继续 Send 可能失败。此时要停止发送并处理错误。

### 4. 批量操作没有错误策略

Client Streaming 经常用于批量上传。一定要提前设计：部分成功是否允许？失败项如何返回？是否需要事务？

---

## 十四、练习任务

1. 发送 5 个订单，确认 `created count=5`。
2. 不发送任何订单，确认 `created count=0`。
3. 发送一个非法订单，观察错误码。
4. 给服务端增加总数量限制，例如最多 100 条。
5. 思考批量导入用户时，如何返回失败明细。

---

## 十五、完成标准

你完成本节后，应该能：

```text
写出 Client Streaming proto
实现服务端循环 Recv
正确处理 io.EOF
使用 SendAndClose 返回汇总响应
客户端循环 Send
客户端使用 CloseAndRecv 结束发送并接收响应
解释 Client Streaming 适合上传、批量导入、采集数据汇总等场景
```

---

## 教程闭环检查

1. **完整操作步骤**：补齐 `CreateOrders` server 和 client。
2. **完整代码**：使用正文中的 `CreateOrders` 和 `createOrders` 完成 demo。
3. **运行命令**：运行 order server 和 order client。
4. **预期输出**：客户端应看到发送日志和 `created count=3`。
5. **常见错误排查**：重点检查 `CloseAndRecv`、`SendAndClose`、`io.EOF` 和非法请求策略。
6. **练习任务**：完成空流、非法请求、数量限制练习。
7. **完成标准**：能独立解释 Client Streaming 的发送完成和汇总返回流程。