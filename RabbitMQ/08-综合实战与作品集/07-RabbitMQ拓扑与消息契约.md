# 07. RabbitMQ 拓扑与消息契约

## 1. 本节目标

定义项目中的：

- exchange。
- queue。
- binding。
- routing key。
- 消息 JSON 契约。

这些设计要写进：

```text
docs/rabbitmq-topology.md
```

## 2. Exchange 设计

```text
order.events                topic
inventory.events            topic
notification.task.direct    direct
order.delay.exchange        direct
order.task.exchange         direct
app.retry.exchange          direct
app.dlx                     direct
```

## 3. Queue 设计

订单事件队列：

```text
order.created.inventory.queue
order.paid.points.queue
order.notify.queue
order.all.audit.queue
```

超时任务：

```text
order.timeout.15s.delay.queue
order.timeout.check.queue
```

重试和死信：

```text
app.retry.10s.queue
app.dlq
```

学习阶段超时时间用 15 秒，方便演示。

生产思路中可以改为 15 分钟。

## 4. Binding 设计

```text
order.events -- order.created --> order.created.inventory.queue
order.events -- order.paid --> order.paid.points.queue
order.events -- order.created --> order.notify.queue
order.events -- order.paid --> order.notify.queue
order.events -- order.cancelled --> order.notify.queue
order.events -- order.# --> order.all.audit.queue

order.task.exchange -- order.timeout.check --> order.timeout.check.queue
```

## 5. 延迟队列参数

```text
Queue: order.timeout.15s.delay.queue
x-message-ttl: 15000
x-dead-letter-exchange: order.task.exchange
x-dead-letter-routing-key: order.timeout.check
```

## 6. 通用消息结构

```json
{
  "message_id": "msg-xxx",
  "event_type": "order.created",
  "version": 1,
  "occurred_at": "2026-07-04T12:00:00Z",
  "correlation_id": "trace-xxx",
  "payload": {}
}
```

## 7. order.created

```json
{
  "message_id": "msg-order-created-1",
  "event_type": "order.created",
  "version": 1,
  "payload": {
    "order_id": 1,
    "user_id": 1001,
    "amount": 9900,
    "items": [
      {
        "sku_id": 2001,
        "quantity": 2
      }
    ]
  }
}
```

## 8. order.paid

```json
{
  "message_id": "msg-order-paid-1",
  "event_type": "order.paid",
  "version": 1,
  "payload": {
    "order_id": 1,
    "user_id": 1001,
    "paid_amount": 9900
  }
}
```

## 9. order.timeout.check

```json
{
  "message_id": "msg-order-timeout-1",
  "event_type": "order.timeout.check",
  "version": 1,
  "payload": {
    "order_id": 1
  }
}
```

## 10. 拓扑声明建议

项目启动时可以由 `mq.DeclareTopology` 统一声明。

也可以通过脚本或 definitions 管理。

学习项目建议代码声明，便于本地一键运行。

## 11. 本节小结

RabbitMQ 拓扑和消息契约必须文档化。

你要让别人能一眼看懂：

```text
谁发布什么事件
事件进入哪个 exchange
哪些队列订阅
消费者如何处理
失败去哪里
```

下一节实现订单 API 和 Outbox 落地。

