# 第 4 阶段：可靠性与一致性

这一阶段是 Kafka 学习的分水岭。会用 API 不难，难的是在真实业务中处理消息丢失、重复消费、顺序性、幂等和一致性。

## 本阶段目标

完成后你应该能做到：

- 解释消息丢失和重复消费分别如何发生。
- 设计 consumer 业务幂等。
- 判断什么时候需要保证顺序。
- 设计 retry topic 和 dead letter topic。
- 理解 Kafka 事务和业务 exactly once 的边界。
- 理解 outbox pattern 的价值。

## 学习顺序

1. [01-message-semantics.md](01-message-semantics.md)
2. [02-idempotent-consumer.md](02-idempotent-consumer.md)
3. [03-retry-and-dlq.md](03-retry-and-dlq.md)
4. [04-transaction-and-outbox.md](04-transaction-and-outbox.md)
5. [05-failure-walkthrough.md](05-failure-walkthrough.md)

## 建议耗时

1 到 2 周。

这部分要慢慢学。它直接决定你能不能把 Kafka 用在订单、支付、库存这类重要链路上。
