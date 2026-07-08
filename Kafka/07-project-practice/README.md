# 第 7 阶段：项目实战

这一阶段要完成一个简化版电商事件驱动后端。重点不是做复杂页面，而是把 Kafka 在 Go 后端中的核心能力串起来。

## 项目目标

构建一个包含订单、库存、支付、通知的事件驱动系统。

完成后你应该能展示：

- Go 服务如何发布 Kafka 事件。
- Consumer group 如何处理业务。
- 如何手动提交 offset。
- 如何做幂等消费。
- 如何设计 retry topic 和 DLQ。
- 如何监控 lag 和处理耗时。
- 如何用 outbox pattern 保证最终一致性。

## 学习顺序

1. [01-requirements-and-architecture.md](01-requirements-and-architecture.md)
2. [02-topic-and-event-design.md](02-topic-and-event-design.md)
3. [03-service-implementation-plan.md](03-service-implementation-plan.md)
4. [04-reliability-checklist.md](04-reliability-checklist.md)
5. [05-acceptance-tests.md](05-acceptance-tests.md)
6. [06-database-and-code-blueprint.md](06-database-and-code-blueprint.md)
7. [07-delivery-structure.md](07-delivery-structure.md)

## 建议耗时

2 周。

项目阶段不要一次写完所有服务。建议每次只打通一条事件链路，确认可观测、可重试、可恢复，再继续扩展。
