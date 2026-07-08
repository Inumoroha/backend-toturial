# 03 Retry Topic 与 Dead Letter Topic

本节目标：设计一套不会卡住主消费链路的失败处理机制。

## 为什么需要 Retry Topic

如果 consumer 处理失败后一直不提交 offset，它会反复读到同一条消息。

这在短暂错误时可以接受，但长期错误会导致：

- 当前 partition 被卡住。
- 后续正常消息无法处理。
- lag 持续升高。
- 服务日志被刷屏。

retry topic 的作用是把失败消息转移到专门的重试链路，让主链路继续前进。

## 推荐 Topic 设计

以 `order.created` 为例：

```text
order.created
order.created.retry.1m
order.created.retry.5m
order.created.retry.30m
order.created.dlq
```

不同重试层级可以由不同 consumer 处理。

## Retry 消息字段

retry 消息必须保留原始上下文：

```json
{
  "event_id": "evt_1",
  "original_topic": "order.created",
  "original_partition": 0,
  "original_offset": 15,
  "attempt": 2,
  "last_error": "database timeout",
  "next_retry_at": "2026-07-05T12:05:00Z",
  "payload": {}
}
```

## 延迟重试怎么做

Kafka 本身不是专业延迟队列，但可以用这些方式：

- 多级 retry topic，加上 consumer 暂停或定时检查。
- retry 消息带 `next_retry_at`，未到时间就暂停或重新投递。
- 使用外部调度系统。
- 使用专门的延迟队列中间件。

初学项目中，多级 retry topic 就足够。

## DLQ 的作用

DLQ 保存最终无法处理的消息。

进入 DLQ 的典型原因：

- JSON 无法解析。
- schema version 不支持。
- 必填字段缺失。
- 重试次数耗尽。
- 业务状态无法修复。

DLQ 不是垃圾桶，而是待处理问题列表。

## DLQ 后续处理

生产环境需要：

- 监控 DLQ 消息数量。
- 告警。
- 提供查询工具。
- 支持修复后重放。
- 记录人工处理日志。

## 本节练习

1. 为 `payment.succeeded` 设计 retry 和 DLQ topic。
2. 写出 retry 消息结构。
3. 写出 DLQ 消息结构。
4. 说明什么情况下原消息可以提交 offset。

