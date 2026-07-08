# 3. Streaming 测试方法

本节目标：学会测试 Server Streaming、Client Streaming 和双向流。

Streaming 测试比 Unary 更复杂，因为它包含多次发送、接收、EOF、取消和中途错误。测试时要明确流的结束条件。

---

## 一、Server Streaming 测试

```go
stream, err := client.ListOrders(ctx, req)
if err != nil {
    t.Fatal(err)
}

var count int
for {
    _, err := stream.Recv()
    if errors.Is(err, io.EOF) {
        break
    }
    if err != nil {
        t.Fatal(err)
    }
    count++
}

if count != 3 {
    t.Fatalf("count=%d", count)
}
```

重点是：正常结束时会收到 `io.EOF`。

---

## 二、Client Streaming 测试

```go
stream, err := client.CreateOrders(ctx)
if err != nil {
    t.Fatal(err)
}

for _, req := range requests {
    if err := stream.Send(req); err != nil {
        t.Fatal(err)
    }
}

resp, err := stream.CloseAndRecv()
if err != nil {
    t.Fatal(err)
}

if resp.CreatedCount != int64(len(requests)) {
    t.Fatalf("created=%d", resp.CreatedCount)
}
```

---

## 三、双向流测试

双向流要特别注意避免死锁。简单场景可以按“发送一条，接收一条”的顺序测试；复杂场景再用 goroutine。

```go
stream, err := client.ChatOrderStatus(ctx)
if err != nil {
    t.Fatal(err)
}

if err := stream.Send(&orderv1.OrderStatusMessage{OrderId: 1, Text: "hello"}); err != nil {
    t.Fatal(err)
}

msg, err := stream.Recv()
if err != nil {
    t.Fatal(err)
}
_ = msg
```

---

## 四、常见问题

- 测试没有超时：流没结束时会一直卡住。
- 不处理 EOF：正常结束被当成失败。
- 双向流测试读写顺序不清：容易死锁。
- 只测试正常流，不测试中途取消。

---

## 五、练习任务

1. 测试 ListOrders 返回 3 条数据。
2. 测试 CreateOrders 汇总数量。
3. 测试双向流收到 ack。
4. 测试 context cancel 后 stream 退出。

---

## 六、完成标准

- 三类 streaming 都有测试。
- EOF 和错误处理清楚。
- 测试不会无限阻塞。


---

## 七、完整操作步骤

本节目标：为第 4 阶段的 streaming RPC 写测试，覆盖 Server Streaming、Client Streaming、Bidirectional Streaming。

操作步骤：

1. 使用 bufconn 启动 OrderService。
2. 测试 `ListOrders` 能读到多条消息并最终 EOF。
3. 测试 `CreateOrders` 发送多条消息后收到汇总结果。
4. 测试 `ChatOrderStatus` 发送消息并收到 ack。
5. 给每个测试设置 context timeout，避免测试卡死。
6. 测试客户端取消 stream。

---

## 八、完整代码：OrderService 测试环境

```go
func newOrderTestEnv(t *testing.T) orderv1.OrderServiceClient {
    t.Helper()

    lis := bufconn.Listen(bufSize)
    server := grpc.NewServer()
    orderv1.RegisterOrderServiceServer(server, newOrderServer())

    go func() {
        if err := server.Serve(lis); err != nil {
            t.Logf("order server stopped: %v", err)
        }
    }()

    dialer := func(context.Context, string) (net.Conn, error) {
        return lis.Dial()
    }

    conn, err := grpc.DialContext(
        context.Background(),
        "bufnet",
        grpc.WithContextDialer(dialer),
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        t.Fatalf("dial: %v", err)
    }

    t.Cleanup(func() {
        _ = conn.Close()
        server.Stop()
        _ = lis.Close()
    })

    return orderv1.NewOrderServiceClient(conn)
}
```

---

## 九、完整代码：Server Streaming 测试

```go
func TestListOrders(t *testing.T) {
    client := newOrderTestEnv(t)

    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    stream, err := client.ListOrders(ctx, &orderv1.ListOrdersRequest{UserId: 100})
    if err != nil {
        t.Fatalf("ListOrders: %v", err)
    }

    var count int
    for {
        _, err := stream.Recv()
        if errors.Is(err, io.EOF) {
            break
        }
        if err != nil {
            t.Fatalf("Recv: %v", err)
        }
        count++
    }

    if count != 3 {
        t.Fatalf("count=%d", count)
    }
}
```

重点：`io.EOF` 是正常结束。

---

## 十、完整代码：Client Streaming 测试

```go
func TestCreateOrders(t *testing.T) {
    client := newOrderTestEnv(t)

    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    stream, err := client.CreateOrders(ctx)
    if err != nil {
        t.Fatalf("CreateOrders: %v", err)
    }

    requests := []*orderv1.CreateOrderRequest{
        {UserId: 100, ProductName: "Book", Quantity: 1},
        {UserId: 100, ProductName: "Keyboard", Quantity: 1},
        {UserId: 100, ProductName: "Mouse", Quantity: 1},
    }

    for _, req := range requests {
        if err := stream.Send(req); err != nil {
            t.Fatalf("Send: %v", err)
        }
    }

    resp, err := stream.CloseAndRecv()
    if err != nil {
        t.Fatalf("CloseAndRecv: %v", err)
    }

    if resp.GetCreatedCount() != 3 {
        t.Fatalf("created=%d", resp.GetCreatedCount())
    }
}
```

重点：发送完必须 `CloseAndRecv()`。

---

## 十一、完整代码：双向流测试

```go
func TestChatOrderStatus(t *testing.T) {
    client := newOrderTestEnv(t)

    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    stream, err := client.ChatOrderStatus(ctx)
    if err != nil {
        t.Fatalf("ChatOrderStatus: %v", err)
    }

    msg := &orderv1.OrderStatusMessage{OrderId: 1, Text: "paid"}
    if err := stream.Send(msg); err != nil {
        t.Fatalf("Send: %v", err)
    }

    ack, err := stream.Recv()
    if err != nil {
        t.Fatalf("Recv: %v", err)
    }

    if ack.GetText() != "ack: paid" {
        t.Fatalf("ack=%q", ack.GetText())
    }

    if err := stream.CloseSend(); err != nil {
        t.Fatalf("CloseSend: %v", err)
    }
}
```

学习阶段先使用“发一条，收一条”的顺序，避免死锁。

---

## 十二、运行命令

```powershell
go test ./... -run 'TestListOrders|TestCreateOrders|TestChatOrderStatus' -v
```

预期输出：

```text
--- PASS: TestListOrders
--- PASS: TestCreateOrders
--- PASS: TestChatOrderStatus
PASS
```

---

## 十三、常见错误排查

### 1. 测试卡死

streaming 测试必须设置 timeout：

```go
context.WithTimeout(...)
```

### 2. Server Streaming 不处理 EOF

如果不处理 `io.EOF`，正常结束会被当成错误。

### 3. Client Streaming 忘记 CloseAndRecv

服务端会一直等客户端继续发送。

### 4. 双向流双方都 Recv

如果客户端和服务端都先 Recv，就会死锁。先设计清楚谁先 Send。

---

## 十四、练习任务

1. 测试 `ListOrders` 返回数量。
2. 测试 `ListOrders` user_id 不存在时 count=0。
3. 测试 `CreateOrders` 空流返回 created=0。
4. 测试 `CreateOrders` 非法请求返回 InvalidArgument。
5. 测试双向流发送 3 条消息并收到 3 条 ack。
6. 测试 context cancel 后 stream 退出。

---

## 十五、完成标准

完成本节后，你应该能：

```text
用 bufconn 测试 Server Streaming
正确断言 io.EOF
用 bufconn 测试 Client Streaming
正确使用 CloseAndRecv
用 bufconn 测试 Bidirectional Streaming
避免 streaming 测试死锁
为 streaming 测试设置 timeout
```

---

## 教程闭环检查

1. **完整操作步骤**：完成三类 streaming 测试和取消测试。
2. **完整代码**：使用正文中的 Order test env 和三个测试函数。
3. **运行命令**：执行 go test -run streaming 相关测试。
4. **预期输出**：三个 streaming 测试 PASS。
5. **常见错误排查**：重点检查 timeout、EOF、CloseAndRecv、双向流死锁。
6. **练习任务**：完成空流、非法请求、多 ack、cancel 测试。
7. **完成标准**：能自动化验证 gRPC streaming 行为。