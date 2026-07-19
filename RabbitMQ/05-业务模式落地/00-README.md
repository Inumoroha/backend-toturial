# 第 5 阶段：业务模式落地

> 本阶段目标：把 RabbitMQ 放进真实 Go 后端业务中。你要学会设计通知系统、订单事件系统、延迟任务、Transactional Outbox、幂等消费者和最终一致性流程，而不只是会写 producer 和 consumer。

## 学习顺序

请按下面顺序学习：

1. [01-本阶段目标与业务落地思维.md](./01-本阶段目标与业务落地思维.md)
2. [02-从技术消息到业务事件.md](./02-从技术消息到业务事件.md)
3. [03-通知系统实战设计.md](./03-通知系统实战设计.md)
4. [04-订单事件系统实战设计.md](./04-订单事件系统实战设计.md)
5. [05-Transactional-Outbox-事务性发件箱.md](./05-Transactional-Outbox-事务性发件箱.md)
6. [06-Outbox-Worker-Go实现思路.md](./06-Outbox-Worker-Go实现思路.md)
7. [07-幂等消费者业务落地.md](./07-幂等消费者业务落地.md)
8. [08-延迟任务与订单超时取消.md](./08-延迟任务与订单超时取消.md)
9. [09-Saga与最终一致性入门.md](./09-Saga与最终一致性入门.md)
10. [10-RabbitMQ-Go项目封装建议.md](./10-RabbitMQ-Go项目封装建议.md)
11. [11-集成测试与本地演练.md](./11-集成测试与本地演练.md)
12. [12-综合项目-Go电商异步订单系统.md](./12-综合项目-Go电商异步订单系统.md)
13. [13-第5阶段练习与自测.md](./13-第5阶段练习与自测.md)

## 建议学习时间

建议用 12 到 18 天完成：

- 第 1 到 2 天：理解业务事件建模。
- 第 3 到 4 天：设计通知系统。
- 第 5 到 6 天：设计订单事件系统。
- 第 7 到 9 天：学习 Transactional Outbox 和 Outbox Worker。
- 第 10 天：落地幂等消费者。
- 第 11 天：实现延迟任务和订单超时取消。
- 第 12 到 13 天：学习 Saga 和最终一致性。
- 第 14 到 16 天：整理项目封装、集成测试和故障演练。
- 第 17 到 18 天：完成综合项目和自测。

## 本阶段你要掌握什么

完成本阶段后，你应该能做到：

- 判断一个业务动作适合 task 还是 event。
- 设计通知系统的 exchange、queue、routing key。
- 设计订单事件系统的 topic exchange。
- 解释数据库事务成功但消息发布失败的问题。
- 使用 Transactional Outbox 解决本地事务和消息发布的一致性。
- 设计 Outbox Worker。
- 设计业务幂等消费者。
- 使用 TTL + DLX 实现订单超时取消。
- 理解 Saga 和最终一致性。
- 把 RabbitMQ 封装进 Go 项目结构。
- 为 RabbitMQ 业务流程写集成测试。

## 本阶段重要提醒

RabbitMQ 不能替你解决所有业务一致性问题。

真实业务中，你要同时设计：

```text
数据库事务
消息发布
消息重试
消费者幂等
业务状态机
死信补偿
监控告警
```

这才是一条完整的后端工程链路。

## 推荐资料

- [RabbitMQ Reliability Guide](https://www.rabbitmq.com/docs/reliability)
- [RabbitMQ Dead Letter Exchanges](https://www.rabbitmq.com/docs/dlx)
- [RabbitMQ Time-To-Live and Expiration](https://www.rabbitmq.com/docs/ttl)
- [Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html)
- [Idempotent Consumer Pattern](https://microservices.io/patterns/communication-style/idempotent-consumer.html)
- [Saga Pattern](https://microservices.io/patterns/data/saga.html)

