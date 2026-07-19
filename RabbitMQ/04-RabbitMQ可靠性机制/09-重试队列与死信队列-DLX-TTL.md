# 09. 重试队列与死信队列：DLX + TTL

## 1. 为什么不能简单 requeue

消费失败时，如果直接：

```go
d.Nack(false, true)
```

消息会重新入队，很快再次投递。

如果错误一直存在，就会形成：

```text
失败 -> requeue -> 立刻再消费 -> 再失败 -> 再 requeue
```

这会拖垮消费者。

更好的方式是：

```text
失败 -> 延迟一段时间 -> 再重试
超过最大次数 -> 进入死信队列
```

## 2. DLX 是什么

DLX 是 Dead Letter Exchange，死信交换机。

当消息在队列中变成 dead letter 时，RabbitMQ 会把它发布到配置的 DLX。

常见死信原因：

- 消息被 reject/nack，并且 `requeue=false`。
- 消息过期。
- 队列超过长度限制。
- quorum queue 中消息超过投递限制。

## 3. TTL 是什么

TTL 是 Time-To-Live，过期时间。

可以设置：

- 消息 TTL。
- 队列级 TTL。

重试队列常用队列级 TTL：

```text
消息进入 retry queue
等待 TTL 到期
过期后 dead-letter 回业务 exchange
```

## 4. 推荐重试拓扑

以订单处理为例：

```text
业务 exchange: stage4.order.exchange
业务 queue:    stage4.order.queue
retry exchange: stage4.order.retry.exchange
retry queue:    stage4.order.retry.5s.queue
DLX exchange:   stage4.order.dlx
DLQ queue:      stage4.order.dlq
```

流程：

```text
stage4.order.exchange -- order.created --> stage4.order.queue

消费失败且可重试：
  -> 发布到 retry exchange
  -> 进入 retry queue
  -> 等待 5 秒 TTL
  -> 过期后 dead-letter 回 stage4.order.exchange
  -> 再次进入 stage4.order.queue

超过最大重试次数：
  -> 发布到 dlx
  -> 进入 dlq
```

## 5. 声明重试队列

Go 中声明 retry queue：

```go
_, err := ch.QueueDeclare(
	"stage4.order.retry.5s.queue",
	true,
	false,
	false,
	false,
	amqp.Table{
		"x-message-ttl":             int32(5000),
		"x-dead-letter-exchange":    "stage4.order.exchange",
		"x-dead-letter-routing-key": "order.created",
	},
)
```

含义：

- 消息在 retry queue 中等待 5 秒。
- 过期后被投递到 `stage4.order.exchange`。
- routing key 使用 `order.created`。

## 6. 声明 DLQ

声明死信 exchange 和 queue：

```go
err := ch.ExchangeDeclare("stage4.order.dlx", "direct", true, false, false, false, nil)
if err != nil {
	return err
}

_, err = ch.QueueDeclare("stage4.order.dlq", true, false, false, false, nil)
if err != nil {
	return err
}

err = ch.QueueBind("stage4.order.dlq", "order.dead", "stage4.order.dlx", false, nil)
if err != nil {
	return err
}
```

超过最大重试次数时，可以把消息发布到：

```text
exchange: stage4.order.dlx
routing key: order.dead
```

## 7. retry count 放在哪里

常见方式：

```text
headers["x-retry-count"]
```

读取：

```go
func retryCount(headers amqp.Table) int32 {
	v, ok := headers["x-retry-count"]
	if !ok {
		return 0
	}
	switch n := v.(type) {
	case int32:
		return n
	case int64:
		return int32(n)
	case int:
		return int32(n)
	default:
		return 0
	}
}
```

发布到 retry queue 时增加：

```go
headers := d.Headers
if headers == nil {
	headers = amqp.Table{}
}
headers["x-retry-count"] = retryCount(headers) + 1
```

## 8. 为什么不直接 nack 到 retry queue

`Nack` 只能告诉 RabbitMQ：

```text
重新入队或不重新入队
```

它不能直接指定：

```text
请把这条消息送到某个 retry exchange。
```

所以常见做法是消费者自己发布到 retry exchange，然后 ack 原消息：

```text
消费失败
  -> 发布到 retry exchange
  -> 发布成功后 ack 原消息
```

注意：这中间也有可靠性问题，生产级方案需要 publisher confirm 或事务性 outbox 配合。第 5 阶段会继续深入业务一致性。

## 9. 简化失败处理流程

伪代码：

```go
if err := handleMessage(d); err != nil {
	count := retryCount(d.Headers)
	if count >= 3 {
		publishToDLQ(d)
		_ = d.Ack(false)
		return
	}

	publishToRetry(d, count+1)
	_ = d.Ack(false)
	return
}

_ = d.Ack(false)
```

核心原则：

```text
只有确认失败消息已经转移到 retry 或 DLQ 后，才 ack 原消息。
```

生产级代码里，`publishToRetry` 和 `publishToDLQ` 应该使用 publisher confirm。

## 10. 死信队列需要监控

DLQ 不是垃圾桶。

进入 DLQ 代表：

```text
自动处理失败，需要排查或人工补偿。
```

必须监控：

- DLQ 消息数量。
- DLQ 增长速率。
- 死信原因。
- message id。
- correlation id。
- 业务 payload。

## 11. 本节小结

你要记住：

- 不要无限 requeue。
- DLX 用来接收死信。
- TTL 可以实现延迟重试。
- retry queue 常用 TTL + dead-letter 回业务 exchange。
- 超过最大重试次数进入 DLQ。
- 转发失败消息到 retry/DLQ 后再 ack 原消息。
- retry/DLQ 也需要 publisher confirm 和监控。

下一节做一个可靠 worker 综合示例。

