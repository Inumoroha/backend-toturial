# 04 Kafka 事务与 Outbox Pattern

本节目标：理解 Kafka 事务解决什么问题，以及为什么 Go 后端项目常常还需要 outbox pattern。

## Kafka 事务解决什么

Kafka 事务适合处理 consume-transform-produce 场景。

例如：

```text
从 topic A 消费消息
处理后写入 topic B
同时提交 topic A 的 offset
```

事务可以让“写入 B”和“提交 A 的 offset”在 Kafka 内部保持原子性。

## Kafka 事务不解决什么

Kafka 事务不能自动解决数据库和 Kafka 之间的一致性。

典型问题：

```text
订单写入数据库成功
发送 Kafka 事件失败
```

或者：

```text
Kafka 事件发送成功
数据库事务回滚
```

这类问题发生在数据库和 Kafka 两个系统之间，不能只靠 Kafka 事务解决。

## Outbox Pattern

outbox pattern 的核心思想：

业务数据和待发送事件写入同一个数据库事务。

流程：

1. 开启数据库事务。
2. 写订单表。
3. 写 outbox_events 表。
4. 提交事务。
5. 后台 worker 扫描 outbox_events。
6. worker 发布 Kafka。
7. 发布成功后标记 outbox event 已发送。

## Outbox 表结构

示例：

```sql
CREATE TABLE outbox_events (
  id VARCHAR(128) PRIMARY KEY,
  aggregate_type VARCHAR(64) NOT NULL,
  aggregate_id VARCHAR(128) NOT NULL,
  event_type VARCHAR(128) NOT NULL,
  payload JSON NOT NULL,
  status VARCHAR(32) NOT NULL,
  retry_count INT NOT NULL DEFAULT 0,
  next_retry_at TIMESTAMP NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  sent_at TIMESTAMP NULL
);
```

## Outbox 的优点

- 避免数据库成功但事件丢失。
- 可以重试发送 Kafka。
- 可以审计事件发布历史。
- 与普通关系型数据库事务结合自然。

## Outbox 的代价

- 多一张表。
- 多一个后台 worker。
- 事件可能不是实时发送，而是最终一致。
- 需要清理历史 outbox 数据。

## Go 项目建议

关键业务推荐：

```text
HTTP handler -> DB transaction 写业务表 + outbox 表 -> outbox worker -> Kafka
```

普通非关键日志可以直接发送 Kafka。

## 本节练习

1. 写出创建订单时使用 outbox 的伪代码。
2. 设计 outbox worker 的重试策略。
3. 思考：outbox event 发送 Kafka 成功，但更新 status 失败，会发生什么？
4. 思考：这种情况下为什么 consumer 幂等仍然必要？

