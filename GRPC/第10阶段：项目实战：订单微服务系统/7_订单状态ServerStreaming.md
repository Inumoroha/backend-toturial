# 7. 订单状态 Server Streaming

本节目标：用 Server Streaming 实现订单状态推送。

订单创建后，客户端可能希望持续观察状态变化。这里用 Server Streaming 实现 `WatchOrderStatus`，让服务端向客户端推送状态事件。

---

## 一、接口设计

```proto
message WatchOrderStatusRequest {
  int64 order_id = 1;
}

message OrderStatusEvent {
  int64 order_id = 1;
  OrderStatus status = 2;
  string message = 3;
}

service OrderService {
  rpc WatchOrderStatus(WatchOrderStatusRequest) returns (stream OrderStatusEvent);
}
```

---

## 二、服务端简化实现

```go
func (s *orderServer) WatchOrderStatus(req *orderv1.WatchOrderStatusRequest, stream orderv1.OrderService_WatchOrderStatusServer) error {
    if req.OrderId <= 0 {
        return status.Error(codes.InvalidArgument, "order id must be positive")
    }

    statuses := []orderv1.OrderStatus{
        orderv1.OrderStatus_ORDER_STATUS_CREATED,
        orderv1.OrderStatus_ORDER_STATUS_PAID,
    }

    for _, st := range statuses {
        select {
        case <-stream.Context().Done():
            return stream.Context().Err()
        case <-time.After(time.Second):
            err := stream.Send(&orderv1.OrderStatusEvent{
                OrderId: req.OrderId,
                Status:  st,
                Message: st.String(),
            })
            if err != nil {
                return err
            }
        }
    }

    return nil
}
```

---

## 三、客户端读取

```go
stream, err := client.WatchOrderStatus(ctx, &orderv1.WatchOrderStatusRequest{OrderId: 1})
if err != nil {
    log.Fatal(err)
}

for {
    event, err := stream.Recv()
    if errors.Is(err, io.EOF) {
        break
    }
    if err != nil {
        log.Fatal(err)
    }
    log.Printf("status=%s message=%s", event.Status, event.Message)
}
```

---

## 四、真实项目中的事件来源

学习项目可以用固定状态或内存 channel。真实项目可能来自：

- 数据库状态变更。
- 消息队列。
- 事件总线。
- 订单状态机。

---

## 五、常见问题

- 客户端断开后服务端还在发送：必须检查 context。
- 没有事件缓存：客户端晚连接可能错过历史状态，真实项目要设计事件存储。
- 状态流没有结束语义：要明确何时结束。
- 把所有订单状态都广播给所有客户端：要按订单或用户隔离。

---

## 六、练习任务

1. 实现 WatchOrderStatus。
2. client 打印状态变化。
3. client 提前取消，观察服务端退出。
4. 为不存在订单返回 NotFound。

---

## 七、完成标准

- 状态流能连续推送。
- 客户端能正常读到 EOF 或取消。
- 服务端不会泄漏 goroutine。

---

## 八、完整操作步骤：给订单增加状态流

Server Streaming 很适合“客户端发起一次请求，服务端持续返回多个事件”的场景。订单状态流可以用于演示：订单创建后，服务端按顺序推送 created、paid、shipped、completed。

实现顺序：

1. 在 proto 中确认 `WatchOrderStatus` 是 server streaming。
2. 在 OrderService 中实现 `WatchOrderStatus`。
3. 先检查订单是否存在，不存在返回 `NotFound`。
4. 准备状态事件切片。
5. 循环 `stream.Send(event)`。
6. 每次发送前检查 `stream.Context().Done()`。
7. 客户端取消后服务端及时退出。

这个功能不要求真实状态机，只要求你掌握 streaming 的服务端写法和取消处理。

---

## 九、完整代码：WatchOrderStatus 实现

在 `internal/app/order/service.go` 中补充：

```go
func (s *Service) WatchOrderStatus(req *orderv1.WatchOrderStatusRequest, stream orderv1.OrderService_WatchOrderStatusServer) error {
	if req.GetOrderId() <= 0 {
		return status.Error(codes.InvalidArgument, "order_id must be positive")
	}

	s.mu.RLock()
	_, ok := s.orders[req.GetOrderId()]
	s.mu.RUnlock()
	if !ok {
		return status.Error(codes.NotFound, "order not found")
	}

	events := []*orderv1.OrderStatusEvent{
		{OrderId: req.GetOrderId(), Status: orderv1.OrderStatus_ORDER_STATUS_CREATED, Message: "order created"},
		{OrderId: req.GetOrderId(), Status: orderv1.OrderStatus_ORDER_STATUS_PAID, Message: "payment confirmed"},
		{OrderId: req.GetOrderId(), Status: orderv1.OrderStatus_ORDER_STATUS_SHIPPED, Message: "package shipped"},
		{OrderId: req.GetOrderId(), Status: orderv1.OrderStatus_ORDER_STATUS_COMPLETED, Message: "order completed"},
	}

	for _, event := range events {
		select {
		case <-stream.Context().Done():
			return stream.Context().Err()
		case <-time.After(500 * time.Millisecond):
		}

		if err := stream.Send(event); err != nil {
			return err
		}
	}

	return nil
}
```

注意这段代码用了 `time.After`，所以文件 import 里需要包含：

```go
import "time"
```

如果前面的 CreateOrder 已经引入了 `time`，就不需要重复添加。

---

## 十、运行命令与预期输出

先创建一个订单：

```powershell
grpcurl -plaintext -d '{\"userId\":1,\"productId\":1,\"quantity\":1,\"requestId\":\"stream-001\"}' localhost:50053 order.v1.OrderService/CreateOrder
```

假设返回订单 ID 是 1，监听状态：

```powershell
grpcurl -plaintext -d '{\"orderId\":1}' localhost:50053 order.v1.OrderService/WatchOrderStatus
```

预期输出会分多次出现：

```json
{
  "orderId": "1",
  "status": "ORDER_STATUS_CREATED",
  "message": "order created"
}
{
  "orderId": "1",
  "status": "ORDER_STATUS_PAID",
  "message": "payment confirmed"
}
{
  "orderId": "1",
  "status": "ORDER_STATUS_SHIPPED",
  "message": "package shipped"
}
{
  "orderId": "1",
  "status": "ORDER_STATUS_COMPLETED",
  "message": "order completed"
}
```

测试不存在订单：

```powershell
grpcurl -plaintext -d '{\"orderId\":999}' localhost:50053 order.v1.OrderService/WatchOrderStatus
```

预期输出包含：

```text
Code: NotFound
Message: order not found
```

---

## 十一、常见错误排查

- stream 一次只返回一个状态：检查 proto 是否写成 `returns (stream OrderStatusEvent)`。
- 客户端取消后服务端还在循环：循环里必须检查 `stream.Context().Done()`。
- 不存在订单也开始推送：发送前先检查订单是否存在。
- 多个客户端互相收到对方订单状态：真实项目需要按 order_id 或 user_id 隔离事件。
- 发送太快看不出 streaming 效果：学习阶段可以加短暂 `time.After`，生产中通常由真实事件驱动。

---

## 十二、教程闭环检查

本篇的完整操作步骤是：确认 proto、实现 WatchOrderStatus、检查订单、循环发送事件、处理客户端取消、用 grpcurl 验证。完整代码包含 server streaming 方法、事件列表、`stream.Send` 和取消检查。运行命令包含创建订单和监听状态流。预期输出覆盖四个状态事件和 NotFound。常见错误排查覆盖 proto 定义、取消处理、订单隔离和发送节奏。练习任务是让客户端在收到 PAID 后主动取消，并观察服务端是否退出。完成标准是：状态流能连续推送，取消后不泄漏 goroutine。
