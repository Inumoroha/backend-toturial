# 第 1 阶段：Kafka 核心概念

这一阶段要把 Kafka 的基本模型吃透。你之后写 Go producer 和 consumer 时遇到的大多数问题，都能回到这些概念里解释。

## 本阶段目标

完成后你应该能做到：

- 解释 broker、topic、partition、replica、leader、offset、consumer group。
- 说明 partition 为什么决定并行度。
- 说明 offset 为什么属于 consumer group。
- 解释 consumer 数量和 partition 数量的关系。
- 能看懂 `kafka-topics.sh --describe` 和 `kafka-consumer-groups.sh --describe` 的主要字段。

## 学习顺序

1. [01-kafka-mental-model.md](01-kafka-mental-model.md)
2. [02-topic-partition-replica.md](02-topic-partition-replica.md)
3. [03-offset-consumer-group.md](03-offset-consumer-group.md)
4. [04-storage-retention-compaction.md](04-storage-retention-compaction.md)
5. [05-core-concept-labs.md](05-core-concept-labs.md)

## 建议耗时

1 周。

这一阶段不要追求参数细节，重点是建立清晰的脑内模型。
