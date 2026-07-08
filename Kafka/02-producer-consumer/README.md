# 第 2 阶段：Producer 与 Consumer

这一阶段开始学习 Kafka 客户端行为。你会看到可靠发送、批量发送、消息 key、手动提交 offset、rebalance、lag 这些真实项目每天都会遇到的问题。

## 本阶段目标

完成后你应该能做到：

- 设计合理的 message key。
- 理解 producer 的可靠性和吞吐参数。
- 判断什么时候可能丢消息。
- 使用手动 offset 提交设计 consumer。
- 理解 rebalance 为什么发生，以及它对 Go 服务的影响。
- 解释 consumer lag 升高时应该如何排查。

## 学习顺序

1. [01-producer-fundamentals.md](01-producer-fundamentals.md)
2. [02-consumer-fundamentals.md](02-consumer-fundamentals.md)
3. [03-rebalance-and-lag.md](03-rebalance-and-lag.md)
4. [04-error-handling-patterns.md](04-error-handling-patterns.md)
5. [05-producer-consumer-labs.md](05-producer-consumer-labs.md)

## 建议耗时

1 到 2 周。

这一阶段建议每个主题都做小实验。不要只记参数名，要知道参数会怎样改变系统行为。
