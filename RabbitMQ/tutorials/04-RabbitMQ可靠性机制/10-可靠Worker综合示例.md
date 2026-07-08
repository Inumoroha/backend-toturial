# 10. 可靠 Worker 综合示例

## 1. 本节目标

把前面学到的机制串起来，设计一个更接近真实项目的 worker。

它包含：

- durable exchange
- durable queue
- persistent message
- manual ack
- prefetch
- publisher confirm
- retry queue
- DLQ
- retry count
- 基础幂等入口

## 2. 示例业务

处理订单创建事件：

```text
event_type: order.created
```

消息流：

```text
Producer
  -> stage4.order.exchange
  -> stage4.order.queue
  -> Worker
```

失败流：

```text
Worker 处理失败
  -> stage4.order.retry.exchange
  -> stage4.order.retry.5s.queue
  -> TTL 到期
  -> stage4.order.exchange
  -> stage4.order.queue
```

超过 3 次：

```text
Worker
  -> stage4.order.dlx
  -> stage4.order.dlq
```

## 3. 拓扑常量

```go
const (
	rabbitURL = "amqp://go_learner:go_learner_pwd@localhost:5672/"

	orderExchange = "stage4.order.exchange"
	orderQueue    = "stage4.order.queue"
	orderKey      = "order.created"

	retryExchange = "stage4.order.retry.exchange"
	retryQueue    = "stage4.order.retry.5s.queue"
	retryKey      = "order.retry"

	dlxExchange = "stage4.order.dlx"
	dlqQueue    = "stage4.order.dlq"
	dlqKey      = "order.dead"

	maxRetries = int32(3)
)
```

## 4. 声明拓扑

```go
func declareTopology(ch *amqp.Channel) error {
	if err := ch.ExchangeDeclare(orderExchange, "direct", true, false, false, false, nil); err != nil {
		return err
	}
	if err := ch.ExchangeDeclare(retryExchange, "direct", true, false, false, false, nil); err != nil {
		return err
	}
	if err := ch.ExchangeDeclare(dlxExchange, "direct", true, false, false, false, nil); err != nil {
		return err
	}

	if _, err := ch.QueueDeclare(orderQueue, true, false, false, false, nil); err != nil {
		return err
	}
	if err := ch.QueueBind(orderQueue, orderKey, orderExchange, false, nil); err != nil {
		return err
	}

	if _, err := ch.QueueDeclare(retryQueue, true, false, false, false, amqp.Table{
		"x-message-ttl":             int32(5000),
		"x-dead-letter-exchange":    orderExchange,
		"x-dead-letter-routing-key": orderKey,
	}); err != nil {
		return err
	}
	if err := ch.QueueBind(retryQueue, retryKey, retryExchange, false, nil); err != nil {
		return err
	}

	if _, err := ch.QueueDeclare(dlqQueue, true, false, false, false, nil); err != nil {
		return err
	}
	return ch.QueueBind(dlqQueue, dlqKey, dlxExchange, false, nil)
}
```

## 5. 可靠发布 helper

生产级代码要考虑批量 confirm，这里先写单条同步 confirm：

```go
func publishWithConfirm(ch *amqp.Channel, confirms <-chan amqp.Confirmation, exchange, key string, msg amqp.Publishing) error {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	if err := ch.PublishWithContext(ctx, exchange, key, true, false, msg); err != nil {
		return err
	}

	select {
	case confirm := <-confirms:
		if !confirm.Ack {
			return fmt.Errorf("message nacked by broker")
		}
		return nil
	case <-time.After(5 * time.Second):
		return fmt.Errorf("publisher confirm timeout")
	}
}
```

注意：

```text
调用这个 helper 前，channel 必须已经 ch.Confirm(false)。
NotifyPublish 也应该提前注册好，并把 confirms channel 传进来复用。
```

## 6. retry count helper

```go
func getRetryCount(headers amqp.Table) int32 {
	if headers == nil {
		return 0
	}
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

func cloneHeaders(headers amqp.Table) amqp.Table {
	copied := amqp.Table{}
	for k, v := range headers {
		copied[k] = v
	}
	return copied
}
```

## 7. 消费失败处理

```go
func handleFailure(ch *amqp.Channel, confirms <-chan amqp.Confirmation, d amqp.Delivery, cause error) {
	count := getRetryCount(d.Headers)
	headers := cloneHeaders(d.Headers)
	headers["x-retry-count"] = count + 1
	headers["x-last-error"] = cause.Error()

	msg := amqp.Publishing{
		ContentType:   d.ContentType,
		DeliveryMode:  amqp.Persistent,
		MessageId:     d.MessageId,
		CorrelationId: d.CorrelationId,
		Type:          d.Type,
		Timestamp:     time.Now(),
		Headers:       headers,
		Body:          d.Body,
	}

	if count >= maxRetries {
		if err := publishWithConfirm(ch, confirms, dlxExchange, dlqKey, msg); err != nil {
			log.Printf("publish to dlq failed: %v", err)
			_ = d.Nack(false, true)
			return
		}
		_ = d.Ack(false)
		log.Printf("message moved to dlq: message_id=%s", d.MessageId)
		return
	}

	if err := publishWithConfirm(ch, confirms, retryExchange, retryKey, msg); err != nil {
		log.Printf("publish to retry failed: %v", err)
		_ = d.Nack(false, true)
		return
	}

	_ = d.Ack(false)
	log.Printf("message moved to retry: message_id=%s retry=%d", d.MessageId, count+1)
}
```

这里有一个重要策略：

```text
如果发布到 retry/DLQ 失败，不 ack 原消息。
```

否则原消息可能丢失。

## 8. 消费主流程

```go
func consume(ch *amqp.Channel) error {
	if err := ch.Confirm(false); err != nil {
		return err
	}
	confirms := ch.NotifyPublish(make(chan amqp.Confirmation, 10))

	if err := ch.Qos(5, 0, false); err != nil {
		return err
	}

	msgs, err := ch.Consume(orderQueue, "stage4-order-worker", false, false, false, false, nil)
	if err != nil {
		return err
	}

	for d := range msgs {
		if err := handleBusiness(d); err != nil {
			handleFailure(ch, confirms, d, err)
			continue
		}

		if err := d.Ack(false); err != nil {
			log.Printf("ack failed: %v", err)
			continue
		}
		log.Printf("message handled: message_id=%s", d.MessageId)
	}
	return nil
}
```

## 9. 业务处理中的幂等入口

```go
func handleBusiness(d amqp.Delivery) error {
	if d.MessageId == "" {
		return fmt.Errorf("missing message_id")
	}

	// 真实项目中，这里应该做数据库幂等检查。
	// 例如 idempotency_key = event_type + business_id。
	if string(d.Body) == "fail" {
		return fmt.Errorf("simulated failure")
	}

	log.Printf("business handled: %s", string(d.Body))
	return nil
}
```

学习阶段先模拟失败。

真实项目中要把：

```text
幂等检查
业务写入
处理记录
```

放进数据库事务里。

## 10. 这个示例还有哪些不足

它已经比普通 demo 更可靠，但还不是完整生产 SDK。

仍需改进：

- connection/channel 断线重连。
- publisher confirm 批量化。
- return 消息异步处理。
- retry/DLQ 发布和原消息 ack 之间的强一致问题。
- 指标监控。
- 结构化日志。
- 数据库幂等实现。
- 配置化 retry 间隔。
- 多级 retry queue。

后续阶段会在业务项目中继续完善。

## 11. 本节小结

可靠 worker 的核心流程：

```text
成功 -> ack
失败且未超重试 -> publish retry confirm -> ack 原消息
失败且超过重试 -> publish dlq confirm -> ack 原消息
retry/dlq 发布失败 -> nack requeue 原消息
```

你要理解这个流程背后的原则，而不是只背代码。

下一节做故障演练与排查。
