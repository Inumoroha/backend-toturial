# 04 存储、Retention 与 Log Compaction

本节目标：理解 Kafka 如何存储消息，以及消息什么时候会被删除或压缩。

## Kafka 是追加写日志

Kafka 的 partition 底层可以理解为一段不断追加的日志。

producer 写入消息时，不是随机写入文件中间，而是追加到末尾。

顺序写对磁盘非常友好，这是 Kafka 高吞吐的重要原因之一。

## Segment

partition 的日志不会永远写在一个巨大文件里，而是分成多个 segment。

这样做的好处：

- 便于删除过期数据。
- 便于查找 offset。
- 便于管理大文件。

你初学时不需要记住所有文件格式，但要理解 Kafka 的消息是持久化在磁盘上的，不是只在内存里。

## Retention

retention 决定消息保留多久或保留多大。

常见配置：

- `retention.ms`
- `retention.bytes`

例如：

```bash
kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --alter \
  --entity-type topics \
  --entity-name order.created \
  --add-config retention.ms=3600000
```

这表示消息保留 1 小时左右。

## 消息被消费后会删除吗

不会因为被消费而立即删除。

Kafka 删除消息主要看 topic 的保留策略，而不是 consumer 是否已经读过。

这正是 Kafka 支持多个 consumer group 独立消费的原因。

## Log Compaction

log compaction 是另一种清理策略。

如果 topic 开启 compaction，Kafka 会按 key 保留较新的值，旧值可能被清理。

适合场景：

- 用户最新状态。
- 商品最新价格。
- 配置变更。
- 数据库变更事件。

不适合场景：

- 需要保留完整历史的订单事件。
- 审计日志。
- 支付流水。

## Retention 和 Compaction 的区别

| 策略 | 关注点 | 适合场景 |
| --- | --- | --- |
| retention | 时间或空间 | 日志、事件历史、行为数据 |
| compaction | 每个 key 的最新值 | 状态同步、配置、最新快照 |

## Go 后端设计建议

- 业务事件 topic 通常使用 retention。
- 状态快照 topic 可以考虑 compaction。
- 支付、订单、审计类消息不要轻易只保留最新值。
- 生产环境不要无脑设置无限保留，要结合磁盘容量规划。

## 本节练习

1. 创建一个 topic，设置较短的 retention。
2. 写入几条消息。
3. 等待超过 retention 时间后再消费，观察消息是否还存在。
4. 思考：如果 consumer 长时间宕机，retention 到期后会发生什么？

