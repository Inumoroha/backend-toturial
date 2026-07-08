# 5. Outbox Pattern

本节目标：理解 outbox pattern 如何解决数据库写入和 Kafka 事件发布之间的最终一致性问题，并能在 Go 后端项目中设计 outbox 表和 worker。

在真实后端系统中，很多业务流程是：

```text
写数据库
发送 Kafka 事件
```

这两个操作跨越两个系统，不能放进同一个普通数据库事务。Outbox pattern 就是解决这个问题的常用方法。

---

## 一、没有 Outbox 会发生什么

创建订单：

```text
写 orders 表成功
发送 order.created 失败
```

结果：

```text
订单存在
下游不知道订单创建
```

反过来：

```text
发送 Kafka 成功
数据库事务回滚
```

结果：

```text
下游收到一个实际不存在的订单事件
```

---

## 二、Outbox 的核心思想

把业务数据和待发送事件写进同一个数据库事务。

流程：

```text
begin tx
写 orders
写 outbox_events
commit tx
outbox worker 扫描 outbox_events
发送 Kafka
发送成功后标记 sent
```

这样保证：

```text
只要订单创建成功，待发送事件一定存在数据库里
```

---

## 三、Outbox 表结构

```sql
CREATE TABLE outbox_events (
  id VARCHAR(128) PRIMARY KEY,
  aggregate_type VARCHAR(64) NOT NULL,
  aggregate_id VARCHAR(128) NOT NULL,
  event_type VARCHAR(128) NOT NULL,
  topic VARCHAR(128) NOT NULL,
  message_key VARCHAR(128) NOT NULL,
  payload JSONB NOT NULL,
  status VARCHAR(32) NOT NULL,
  retry_count INT NOT NULL DEFAULT 0,
  next_retry_at TIMESTAMP NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  sent_at TIMESTAMP NULL
);
```

索引：

```sql
CREATE INDEX idx_outbox_status_next_retry
ON outbox_events(status, next_retry_at);
```

---

## 四、状态设计

常见状态：

```text
PENDING
SENDING
SENT
FAILED
```

简单项目也可以只用：

```text
PENDING
SENT
```

但生产中需要考虑：

- worker 并发抢任务。
- 发送失败重试。
- 卡在 SENDING 的恢复。

---

## 五、创建订单时写 Outbox

伪代码：

```go
tx := db.BeginTx(ctx)
defer tx.Rollback()

insertOrder(tx, order)
insertOrderItems(tx, items)

event := BuildOrderCreatedEvent(order)
insertOutbox(tx, OutboxEvent{
    ID:            event.EventID,
    AggregateType: "order",
    AggregateID:   order.ID,
    EventType:     "order.created",
    Topic:         "order.created",
    MessageKey:    order.ID,
    Payload:       event,
    Status:        "PENDING",
})

return tx.Commit()
```

关键：

```text
orders 和 outbox_events 在同一个事务里
```

---

## 六、Outbox Worker

伪代码：

```go
for ctx.Err() == nil {
    events := repo.LockPendingEvents(ctx, 100)

    for _, event := range events {
        err := producer.Publish(ctx, event.Topic, Message{
            Key: event.MessageKey,
            Value: event.Payload,
        })
        if err != nil {
            repo.MarkRetry(ctx, event.ID, err)
            continue
        }

        repo.MarkSent(ctx, event.ID)
    }

    time.Sleep(time.Second)
}
```

多 worker 时，PostgreSQL 可使用：

```sql
FOR UPDATE SKIP LOCKED
```

避免多个 worker 抢同一条事件。

---

## 七、Outbox 仍然可能重复

场景：

```text
发送 Kafka 成功
更新 outbox status=sent 失败
worker 下次又发送
```

所以 outbox 解决的是：

```text
事件不丢
```

不保证：

```text
事件绝不重复
```

consumer 仍然必须幂等。

---

## 八、Outbox 的代价

优点：

- 数据库写入和事件发布最终一致。
- 事件可重试。
- 事件可审计。

代价：

- 多一张表。
- 多一个 worker。
- 事件不是立刻发送，而是最终发送。
- 需要清理历史 outbox。
- 需要监控积压。

---

## 九、监控指标

需要监控：

- PENDING 数量。
- FAILED 数量。
- 最老 PENDING age。
- worker 发送成功数。
- worker 发送失败数。

如果 outbox 积压持续增长，下游 Kafka 或 worker 可能有问题。

---

## 十、常见误区

### 1. Outbox 能保证不重复

不能。

consumer 仍要幂等。

### 2. Outbox 可以不监控

不行。

积压会导致事件延迟。

### 3. Outbox worker 发送失败可以直接丢弃

不行。

应该重试并记录失败。

### 4. event_id 每次发送生成

错误。

outbox 里的 event_id 必须稳定。

---

## 十一、本节练习

1. 创建 outbox_events 表。
2. 写出创建订单时写 outbox 的伪代码。
3. 写出 outbox worker 主循环。
4. 解释 outbox 解决什么问题。
5. 解释 outbox 为什么仍然需要 consumer 幂等。
6. 设计 outbox 积压告警。

---

## 十二、本节小结

- Outbox 用于解决数据库和 Kafka 的最终一致性。
- 业务表和 outbox 表在同一数据库事务中写入。
- Outbox worker 异步发送 Kafka。
- 发送成功后标记 sent。
- Outbox 防丢，但仍可能重复。
- consumer 幂等仍然必要。
- Outbox 积压必须监控。

---

## 十四、Outbox 表最小字段

```sql
CREATE TABLE outbox_events (
  id VARCHAR(128) PRIMARY KEY,
  aggregate_type VARCHAR(64) NOT NULL,
  aggregate_id VARCHAR(128) NOT NULL,
  event_type VARCHAR(128) NOT NULL,
  topic VARCHAR(128) NOT NULL,
  message_key VARCHAR(128) NOT NULL,
  payload JSONB NOT NULL,
  status VARCHAR(32) NOT NULL DEFAULT 'PENDING',
  retry_count INT NOT NULL DEFAULT 0,
  next_retry_at TIMESTAMP NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  sent_at TIMESTAMP NULL
);
```

查询索引：

```sql
CREATE INDEX idx_outbox_status_retry
ON outbox_events(status, next_retry_at, created_at);
```

---

## 十五、面试表达

```text
Outbox 解决的是本地数据库事务和 Kafka 发送之间无法原子提交的问题。
我会把业务数据和 outbox 事件写入同一个数据库事务，提交后由 worker 异步发布。
如果 Kafka 不可用，事件仍保留在 outbox 表中；如果 worker 重复发布，consumer 通过 event_id 做幂等。
```

---

## 十六、Outbox Worker 查询

worker 扫描时可以使用：

```sql
SELECT id, topic, message_key, payload
FROM outbox_events
WHERE status = 'PENDING'
  AND (next_retry_at IS NULL OR next_retry_at <= now())
ORDER BY created_at
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

`SKIP LOCKED` 可以让多个 worker 并发时不互相抢同一批记录。

---

## 十七、Outbox 验收

- [ ] 业务表和 outbox 在同一事务写入。
- [ ] Kafka 不可用时 outbox 记录仍存在。
- [ ] worker 成功发送后标记 SENT。
- [ ] worker 失败时增加 retry_count。
- [ ] 重复发送不会导致 consumer 重复业务副作用。

---

## 十八、Outbox 积压排查

如果 `PENDING` 越来越多，按顺序查：

1. worker 是否运行。
2. Kafka 是否可用。
3. producer 是否有权限。
4. 单条消息是否过大。
5. worker 发送速率是否低于写入速率。

Outbox 表不是垃圾桶。积压必须有指标和告警。

---

## 十九、最终验收

关闭 Kafka 后创建订单，确认 outbox 仍有 `PENDING` 事件。

恢复 Kafka 并启动 worker，确认事件最终变为 `SENT`。

---

## 二十、排查 SQL

```sql
SELECT id, event_type, status, retry_count, created_at, sent_at
FROM outbox_events
ORDER BY created_at DESC
LIMIT 20;
```

这条 SQL 是排查 Outbox 是否积压的第一步。

---

## 二十二、Outbox 上线检查

Outbox 方案上线前，至少确认：

```text
业务数据和 outbox 记录在同一个数据库事务中写入。
outbox 表有状态字段和重试次数字段。
publisher 支持批量扫描和发送。
发送成功后能标记为 published。
发送失败后能记录错误原因。
长期失败的数据有告警和人工处理入口。
重复发送时 consumer 仍然幂等。
```

Outbox 解决的是“业务事务和消息发送之间的原子性缺口”，不是解决所有可靠性问题。

因此它必须和 consumer 幂等、监控告警、重放工具一起使用。
