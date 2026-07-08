# 09. Outbox Worker 与可靠发布

## 1. 本节目标

Outbox Worker 负责把数据库里的 outbox 消息可靠发布到 RabbitMQ。

流程：

```text
扫描 pending outbox
  -> 标记 publishing
  -> PublishWithContext
  -> publisher confirm
  -> 标记 sent
```

## 2. Worker 主循环

```go
func (w *Worker) Run(ctx context.Context) error {
	ticker := time.NewTicker(w.pollInterval)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-ticker.C:
			if err := w.ProcessBatch(ctx); err != nil {
				w.logger.Printf("process outbox batch failed: %v", err)
			}
		}
	}
}
```

## 3. 扫描 pending 消息

建议每批取固定数量：

```sql
SELECT *
FROM outbox_messages
WHERE status = 'pending'
  AND (next_retry_at IS NULL OR next_retry_at <= NOW())
ORDER BY id
LIMIT ?;
```

多 worker 时要使用锁或状态抢占，避免重复处理同一行。

学习阶段可以先单 worker。

## 4. 发布消息

发布字段来自 outbox：

```go
msg := amqp.Publishing{
	ContentType:   "application/json",
	DeliveryMode:  amqp.Persistent,
	MessageId:     outbox.MessageID,
	Type:          outbox.EventType,
	CorrelationId: outbox.CorrelationID,
	Timestamp:     time.Now(),
	Body:          outbox.Payload,
}
```

发布：

```go
err := publisher.Publish(ctx, outbox.ExchangeName, outbox.RoutingKey, msg)
```

Publisher 内部必须：

- 开启 confirm mode。
- `mandatory=true`。
- 监听 return。
- 等待 confirm ack。

## 5. 成功后标记 sent

```sql
UPDATE outbox_messages
SET status = 'sent',
    sent_at = NOW()
WHERE id = ?;
```

如果发布成功但更新 sent 失败，消息可能以后重复发布。

这就是为什么消费者必须幂等。

## 6. 失败后重试

```sql
UPDATE outbox_messages
SET status = 'pending',
    retry_count = retry_count + 1,
    last_error = ?,
    next_retry_at = ?
WHERE id = ?;
```

重试间隔：

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

`dead` 必须告警。

## 7. Outbox Worker 指标

建议记录：

- pending 数量。
- sent 数量。
- publish failed 数量。
- dead 数量。
- 最老 pending 年龄。
- publish confirm 耗时。

最老 pending 年龄尤其重要。

如果它持续增长，说明消息发布链路卡住。

## 8. 本节小结

Outbox Worker 要可靠，必须做到：

- pending 可重试。
- publisher confirm。
- mandatory return。
- sent 状态可追踪。
- failed/dead 可告警。
- 消费者幂等兜底重复发布。

下一节实现各消费者服务。

