# 3. Bidirectional Streaming 双向流

本节目标：掌握双向流的基本模型和并发读写方式。

双向流允许客户端和服务端同时多次发送消息。它更接近一条持续存在的通信通道，适合聊天、实时状态同步、在线协作等场景。

---

## 一、核心直觉

- proto 中请求和响应都加 `stream`。
- 双向流的读写可以相互独立。
- Go 中常用两个 goroutine 分别处理发送和接收。
- 必须处理 context 取消、EOF 和发送错误。
- 双向流比 Unary 更容易出现资源泄漏，必须明确退出条件。

---

## 二、动手步骤

1. 定义 `ChatOrderStatus`。
2. 客户端发送订单状态消息。
3. 服务端收到后返回确认或推送新状态。
4. 客户端一边发送一边接收。
5. 使用 context 控制整体生命周期。

---

## 三、参考代码或命令

```proto
rpc ChatOrderStatus(stream OrderStatusMessage) returns (stream OrderStatusMessage);
```

服务端简化写法：

```go
for {
    msg, err := stream.Recv()
    if errors.Is(err, io.EOF) {
        return nil
    }
    if err != nil {
        return err
    }
    if err := stream.Send(&orderv1.OrderStatusMessage{OrderId: msg.OrderId, Text: "ack: " + msg.Text}); err != nil {
        return err
    }
}
```

---

## 四、常见问题

- 没有退出条件：连接断开后 goroutine 可能泄漏。
- 发送和接收互相阻塞：复杂场景要用 goroutine 和 channel 解耦。
- 把双向流当成万能方案：能用 Unary 或 Server Streaming 解决时，不必上来就双向流。

---

## 五、练习任务

- 实现一个简单的订单状态聊天。
- client 发送 3 条消息后主动关闭。
- 服务端每收到一条回复一条 ack。

---

## 六、完成标准

- 能跑通双向流 demo。
- 能解释双向流的退出机制。
- 知道双向流需要更谨慎地管理资源。


---

## 七、完整操作步骤

Bidirectional Streaming 的目标是：客户端和服务端都可以在同一条 RPC 中多次发送消息。

本节先做一个简单版本：客户端发送一条订单状态消息，服务端立即回复一条 ack。这样不容易死锁，也便于理解双向流的基础模型。

要完成：

1. 确认 proto 中有 `ChatOrderStatus`。
2. 服务端循环 `Recv()`。
3. 服务端每收到一条消息就 `Send()` 一条 ack。
4. 客户端发送多条消息。
5. 客户端接收对应 ack。
6. 客户端主动 `CloseSend()`。

proto 方法：

```proto
rpc ChatOrderStatus(stream OrderStatusMessage) returns (stream OrderStatusMessage);
```

生成的服务端签名类似：

```go
ChatOrderStatus(orderv1.OrderService_ChatOrderStatusServer) error
```

---

## 八、完整服务端代码

```go
func (s *orderServer) ChatOrderStatus(stream orderv1.OrderService_ChatOrderStatusServer) error {
    for {
        msg, err := stream.Recv()
        if errors.Is(err, io.EOF) {
            log.Println("client closed bidirectional stream")
            return nil
        }
        if err != nil {
            return err
        }

        log.Printf("recv from client: order_id=%d text=%s", msg.GetOrderId(), msg.GetText())

        ack := &orderv1.OrderStatusMessage{
            OrderId: msg.GetOrderId(),
            Text:    "ack: " + msg.GetText(),
        }

        if err := stream.Send(ack); err != nil {
            return err
        }
    }
}
```

这个版本是最容易理解的“双向流”：

```text
客户端 Send -> 服务端 Recv
服务端 Send -> 客户端 Recv
客户端 Send -> 服务端 Recv
服务端 Send -> 客户端 Recv
```

---

## 九、完整客户端代码：顺序读写版

```go
func chatOrderStatus(client orderv1.OrderServiceClient) error {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    stream, err := client.ChatOrderStatus(ctx)
    if err != nil {
        return err
    }

    messages := []*orderv1.OrderStatusMessage{
        {OrderId: 1, Text: "client says order 1 paid"},
        {OrderId: 1, Text: "client says order 1 shipped"},
        {OrderId: 1, Text: "client says order 1 completed"},
    }

    for _, msg := range messages {
        log.Printf("send: %s", msg.GetText())
        if err := stream.Send(msg); err != nil {
            return err
        }

        ack, err := stream.Recv()
        if err != nil {
            return err
        }
        log.Printf("recv: %s", ack.GetText())
    }

    return stream.CloseSend()
}
```

这种写法不是最高性能，但非常适合学习：发送和接收顺序清晰，不容易卡住。

---

## 十、进阶客户端：读写分离版

真实聊天或实时协作中，发送和接收通常互不等待。可以用两个 goroutine：

```go
func chatOrderStatusConcurrent(client orderv1.OrderServiceClient) error {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    stream, err := client.ChatOrderStatus(ctx)
    if err != nil {
        return err
    }

    errCh := make(chan error, 2)

    go func() {
        for i := 0; i < 3; i++ {
            msg := &orderv1.OrderStatusMessage{OrderId: 1, Text: fmt.Sprintf("message %d", i+1)}
            if err := stream.Send(msg); err != nil {
                errCh <- err
                return
            }
            time.Sleep(300 * time.Millisecond)
        }
        errCh <- stream.CloseSend()
    }()

    go func() {
        for {
            msg, err := stream.Recv()
            if errors.Is(err, io.EOF) {
                errCh <- nil
                return
            }
            if err != nil {
                errCh <- err
                return
            }
            log.Printf("recv: %s", msg.GetText())
        }
    }()

    for i := 0; i < 2; i++ {
        if err := <-errCh; err != nil {
            return err
        }
    }
    return nil
}
```

这个版本更接近真实双向流，但也更容易写错。初学阶段先掌握顺序读写版。

---

## 十一、运行命令

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
send: client says order 1 paid
recv: ack: client says order 1 paid
send: client says order 1 shipped
recv: ack: client says order 1 shipped
send: client says order 1 completed
recv: ack: client says order 1 completed
```

预期服务端输出：

```text
recv from client: order_id=1 text=client says order 1 paid
recv from client: order_id=1 text=client says order 1 shipped
recv from client: order_id=1 text=client says order 1 completed
client closed bidirectional stream
```

---

## 十二、常见错误排查

### 1. 双方都在 Recv，导致卡住

如果客户端一开始就 `Recv()`，服务端也一开始 `Recv()`，双方都在等对方发送，就会卡住。

解决：设计清楚谁先发，或者使用 goroutine 分离读写。

### 2. 客户端忘记 CloseSend

客户端发送完后应该：

```go
stream.CloseSend()
```

否则服务端可能不知道客户端已经结束发送。

### 3. goroutine 泄漏

读写分离版如果没有 context、没有 EOF 处理、没有错误通道，很容易留下 goroutine。真实项目必须设计退出条件。

### 4. Send 和 Recv 错误被忽略

任何一次 `Send` 或 `Recv` 都可能失败。不要忽略错误。

---

## 十三、什么时候适合双向流

适合：

- 聊天。
- 实时协作。
- 游戏状态同步。
- 客户端和服务端都需要随时发送事件。
- 长连接控制通道。

不适合：

- 普通查询。
- 简单创建订单。
- 一次性上传少量数据。
- 服务端只需要单向推送。

能用 Unary 或 Server Streaming 解决时，不要一上来就用双向流。

---

## 十四、练习任务

1. 使用顺序读写版发送 3 条消息。
2. 故意去掉 `CloseSend()`，观察服务端是否能正常结束。
3. 把 timeout 改成 1 秒，观察超时行为。
4. 尝试读写分离版，确认不会卡住。
5. 为服务端增加空 text 校验，返回 InvalidArgument。

---

## 十五、完成标准

你完成本节后，应该能：

```text
写出 Bidirectional Streaming proto
实现服务端循环 Recv 和 Send
实现客户端 Send/Recv
知道 CloseSend 的作用
知道双向流为什么可能死锁
知道如何用 goroutine 分离读写
知道双向流适合实时双向通信，不适合普通请求响应
```

---

## 教程闭环检查

1. **完整操作步骤**：完成 `ChatOrderStatus` 的 server 和 client。
2. **完整代码**：至少实现顺序读写版，进阶再实现 goroutine 读写分离版。
3. **运行命令**：运行 order server 和 order client。
4. **预期输出**：客户端应看到每条消息对应一条 `ack`。
5. **常见错误排查**：重点检查双方都 Recv、忘记 CloseSend、goroutine 泄漏、忽略 Send/Recv 错误。
6. **练习任务**：完成 timeout、CloseSend、读写分离和非法消息练习。
7. **完成标准**：能独立解释双向流的通信顺序和退出条件。