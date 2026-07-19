# 06. Outbox Worker Go 实现思路

## 1. Outbox Worker 的职责

Outbox Worker 负责：

```text
扫描 outbox_messages 表
发布消息到 RabbitMQ
等待 publisher confirm
标记消息 sent
失败后重试
超过次数标记 dead
```

它是业务数据库和 RabbitMQ 之间的桥。

## 2. 基本循环

```text
每隔 1 秒扫描 pending 消息
  -> 加锁取一批
  -> 标记 publishing
  -> 发布 RabbitMQ
  -> confirm 成功
  -> 标记 sent
  -> 失败则 retry_count + 1
```

伪代码：

```go
for {
	messages := repo.FetchPending(ctx, 100)
	for _, msg := range messages {
		processOne(ctx, msg)
	}
	time.Sleep(time.Second)
}
```

## 3. 取消息时要避免多 worker 重复抢

如果多个 Outbox Worker 同时运行，要避免同时处理同一条 outbox。

常见做法：

- 使用数据库行锁。
- 使用 `SELECT ... FOR UPDATE SKIP LOCKED`。
- 先原子更新状态再处理。

MySQL 8 / PostgreSQL 可以考虑：

```sql
SELECT *
FROM outbox_messages
WHERE status = 'pending'
  AND (next_retry_at IS NULL OR next_retry_at <= NOW())
ORDER BY id
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

然后在同一个事务里标记：

```sql
UPDATE outbox_messages
SET status = 'publishing'
WHERE id = ?;
```

## 4. 发布时使用 publisher confirm

Outbox Worker 发布消息必须等待 confirm。

伪代码：

```go
func publishOutbox(ctx context.Context, ch *amqp.Channel, msg OutboxMessage) error {
	body := []byte(msg.Payload)

	return publishWithConfirm(ch, msg.ExchangeName, msg.RoutingKey, amqp.Publishing{
		ContentType:   "application/json",
		DeliveryMode:  amqp.Persistent,
		MessageId:     msg.MessageID,
		Type:          msg.EventType,
		CorrelationId: msg.CorrelationID,
		Body:          body,
	})
}
```

发布成功后：

```sql
UPDATE outbox_messages
SET status = 'sent',
    sent_at = NOW()
WHERE id = ?;
```

## 5. 失败重试策略

失败时：

```sql
UPDATE outbox_messages
SET status = 'pending',
    retry_count = retry_count + 1,
    last_error = ?,
    next_retry_at = ?
WHERE id = ?;
```

重试间隔可以使用指数退避：

```text
第 1 次：10 秒
第 2 次：30 秒
第 3 次：2 分钟
第 4 次：10 分钟
```

超过最大次数：

```sql
UPDATE outbox_messages
SET status = 'dead',
    last_error = ?
WHERE id = ?;
```

`dead` 状态必须告警。

## 6. publish 成功但标记 sent 失败怎么办

这是 Outbox 的经典边界：

```text
RabbitMQ 发布成功
数据库更新 sent 失败
```

结果：

```text
Outbox Worker 以后可能再次发布同一条消息。
```

所以消费者必须幂等。

这不是 bug，而是 Outbox 模式的正常取舍：

```text
用可能重复，换取消息不丢。
```

## 7. Outbox Worker 目录建议

```text
internal/
  outbox/
    message.go
    repository.go
    worker.go
    publisher.go
  mq/
    connection.go
    publisher.go
```

结构：

```text
repository.go -> 访问 outbox_messages 表
worker.go     -> 扫描和调度
publisher.go  -> RabbitMQ 发布
message.go    -> OutboxMessage 结构体
```

## 8. OutboxMessage 结构体

```go
type OutboxMessage struct {
	ID            int64
	MessageID     string
	AggregateType string
	AggregateID   string
	EventType     string
	ExchangeName  string
	RoutingKey    string
	Payload       []byte
	Headers       map[string]any
	Status        string
	RetryCount    int
	LastError     string
}
```

## 9. Worker 配置

建议配置化：

```go
type WorkerConfig struct {
	BatchSize       int
	PollInterval    time.Duration
	MaxRetries      int
	ConfirmTimeout  time.Duration
	ShutdownTimeout time.Duration
}
```

不要把这些写死在代码里。

## 10. 监控指标

Outbox Worker 需要监控：

- pending 数量。
- publishing 卡住数量。
- sent 速率。
- failed/dead 数量。
- 发布耗时。
- publisher confirm 超时次数。
- 最老 pending 消息年龄。

最老 pending 消息年龄非常重要：

```text
如果最老 pending 已经 10 分钟，说明消息发布链路可能卡住。
```

## 11. 本节小结

Outbox Worker 的关键：

- 扫描 pending。
- 防止多实例重复抢。
- 使用 publisher confirm。
- 成功后标记 sent。
- 失败后延迟重试。
- 超过次数标记 dead。
- 消费者必须幂等。

下一节学习幂等消费者如何真正落地。

