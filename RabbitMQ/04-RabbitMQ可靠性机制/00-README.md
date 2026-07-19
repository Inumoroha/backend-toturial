# 第 4 阶段：RabbitMQ 可靠性机制

> 本阶段目标：从“能发送和消费消息”升级到“能设计可靠消息链路”。你要掌握消费确认、负确认、prefetch、持久化、publisher confirm、不可路由消息处理、幂等消费、重试和死信队列。

## 学习顺序

请按下面顺序学习：

1. [01-本阶段目标与可靠性地图.md](./01-本阶段目标与可靠性地图.md)
2. [02-消费端确认机制-Ack.md](./02-消费端确认机制-Ack.md)
3. [03-Nack-Reject-Requeue-失败处理.md](./03-Nack-Reject-Requeue-失败处理.md)
4. [04-Prefetch-QoS-消费者限流.md](./04-Prefetch-QoS-消费者限流.md)
5. [05-持久化链路-Durable-Persistent.md](./05-持久化链路-Durable-Persistent.md)
6. [06-Publisher-Confirm-生产端确认.md](./06-Publisher-Confirm-生产端确认.md)
7. [07-Mandatory-Publish-与不可路由消息.md](./07-Mandatory-Publish-与不可路由消息.md)
8. [08-消费者幂等设计.md](./08-消费者幂等设计.md)
9. [09-重试队列与死信队列-DLX-TTL.md](./09-重试队列与死信队列-DLX-TTL.md)
10. [10-可靠Worker综合示例.md](./10-可靠Worker综合示例.md)
11. [11-故障演练与排查.md](./11-故障演练与排查.md)
12. [12-第4阶段练习与自测.md](./12-第4阶段练习与自测.md)

## 建议学习时间

建议用 10 到 14 天完成：

- 第 1 天：建立可靠性地图。
- 第 2 天：深入 ack。
- 第 3 天：理解 nack、reject、requeue。
- 第 4 天：学习 prefetch 和消费者限流。
- 第 5 天：学习 durable 和 persistent。
- 第 6 到 7 天：学习 publisher confirm。
- 第 8 天：学习 mandatory publish 和 return。
- 第 9 天：学习消费者幂等。
- 第 10 到 11 天：学习重试队列和死信队列。
- 第 12 到 14 天：完成综合示例、故障演练和自测。

## 本阶段你要掌握什么

完成本阶段后，你应该能做到：

- 解释 auto ack 和 manual ack 的区别。
- 正确使用 `Ack`、`Nack`、`Reject`。
- 避免无限 requeue。
- 使用 `Qos` / prefetch 控制消费者未确认消息数量。
- 解释 durable exchange、durable queue、persistent message 的区别。
- 使用 publisher confirm 确认 broker 接收消息。
- 使用 mandatory publish 发现不可路由消息。
- 设计消费者幂等。
- 设计 retry queue 和 dead letter queue。
- 排查 ready、unacked、死信、重复消费等问题。

## 本阶段重要提醒

RabbitMQ 可靠性不是一个开关，而是一条链路：

```text
生产者确认发布成功
  -> RabbitMQ 正确路由和保存
  -> 消费者成功处理后 ack
  -> 失败时可重试
  -> 毒消息进入死信
  -> 业务侧保证幂等
```

任何一环缺失，都可能导致消息丢失、重复、堆积或难以恢复。

## 官方资料

- [Consumer Acknowledgements and Publisher Confirms](https://www.rabbitmq.com/docs/confirms)
- [Consumer Prefetch](https://www.rabbitmq.com/docs/consumer-prefetch)
- [Dead Letter Exchanges](https://www.rabbitmq.com/docs/dlx)
- [Time-To-Live and Expiration](https://www.rabbitmq.com/docs/ttl)
- [Reliability Guide](https://www.rabbitmq.com/docs/reliability)
- [amqp091-go package docs](https://pkg.go.dev/github.com/rabbitmq/amqp091-go)

