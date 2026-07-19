# 09. 消息结构：JSON、Properties、Headers

## 1. 为什么要设计消息结构

前面的 demo 里有些消息只是普通字符串。

真实后端项目中，消息通常应该是结构化数据：

```json
{
  "message_id": "msg-001",
  "event_type": "order.created",
  "version": 1,
  "occurred_at": "2026-07-04T12:00:00Z",
  "correlation_id": "trace-001",
  "payload": {
    "order_id": 1001,
    "user_id": 2001
  }
}
```

结构化消息能帮助你：

- 做幂等。
- 做日志追踪。
- 做版本兼容。
- 做重试和死信分析。
- 让多个服务理解同一种事件。

## 2. 推荐基础消息模型

在 Go 中可以定义：

```go
type Event struct {
	MessageID     string          `json:"message_id"`
	EventType     string          `json:"event_type"`
	Version       int             `json:"version"`
	OccurredAt    time.Time       `json:"occurred_at"`
	CorrelationID string          `json:"correlation_id"`
	Payload       json.RawMessage `json:"payload"`
}
```

其中：

| 字段 | 作用 |
| --- | --- |
| `message_id` | 消息唯一 ID，用于幂等和排查 |
| `event_type` | 事件类型，例如 `order.created` |
| `version` | 消息结构版本 |
| `occurred_at` | 事件发生时间 |
| `correlation_id` | 链路追踪 ID |
| `payload` | 业务数据 |

## 3. Payload 示例

不同事件可以有不同 payload。

订单创建：

```go
type OrderCreatedPayload struct {
	OrderID int64 `json:"order_id"`
	UserID  int64 `json:"user_id"`
	Amount  int64 `json:"amount"`
}
```

用户注册：

```go
type UserRegisteredPayload struct {
	UserID int64  `json:"user_id"`
	Email  string `json:"email"`
}
```

外层 `Event` 保持统一，内层 payload 按事件变化。

## 4. 发布 JSON 消息

示例：

```go
payload := OrderCreatedPayload{
	OrderID: 1001,
	UserID:  2001,
	Amount:  9900,
}

payloadBytes, err := json.Marshal(payload)
if err != nil {
	return err
}

event := Event{
	MessageID:     "msg-order-1001-created",
	EventType:     "order.created",
	Version:       1,
	OccurredAt:    time.Now().UTC(),
	CorrelationID: "trace-abc-001",
	Payload:       payloadBytes,
}

body, err := json.Marshal(event)
if err != nil {
	return err
}

err = ch.PublishWithContext(
	ctx,
	"stage3.business.topic",
	"order.created",
	false,
	false,
	amqp.Publishing{
		ContentType:   "application/json",
		DeliveryMode:  amqp.Persistent,
		MessageId:     event.MessageID,
		CorrelationId: event.CorrelationID,
		Type:          event.EventType,
		Timestamp:     event.OccurredAt,
		Headers: amqp.Table{
			"schema_version": event.Version,
		},
		Body: body,
	},
)
```

## 5. 消费 JSON 消息

消费者先解析外层事件：

```go
var event Event
if err := json.Unmarshal(d.Body, &event); err != nil {
	log.Printf("invalid event json: %v", err)
	_ = d.Ack(false)
	return
}
```

再按事件类型解析 payload：

```go
switch event.EventType {
case "order.created":
	var payload OrderCreatedPayload
	if err := json.Unmarshal(event.Payload, &payload); err != nil {
		log.Printf("invalid order.created payload: %v", err)
		_ = d.Ack(false)
		return
	}
	log.Printf("order created: order_id=%d user_id=%d", payload.OrderID, payload.UserID)

default:
	log.Printf("unknown event type: %s", event.EventType)
}
```

这一阶段先用 `Ack(false)` 处理格式错误。后续可靠性阶段会讨论什么时候 ack、什么时候 nack、什么时候进入死信。

## 6. Properties 和 Body 的关系

RabbitMQ 消息有两部分：

```text
properties
body
```

Body 是消息主体。

Properties 是消息元数据，例如：

- `ContentType`
- `DeliveryMode`
- `MessageId`
- `CorrelationId`
- `Type`
- `Timestamp`
- `Headers`

在 Go 中就是：

```go
amqp.Publishing{
	ContentType:   "application/json",
	DeliveryMode:  amqp.Persistent,
	MessageId:     "msg-001",
	CorrelationId: "trace-001",
	Type:          "order.created",
	Timestamp:     time.Now(),
	Headers:       amqp.Table{"schema_version": 1},
	Body:          body,
}
```

## 7. 常用 Properties 建议

| 属性 | 建议 |
| --- | --- |
| `ContentType` | JSON 使用 `application/json` |
| `DeliveryMode` | 重要业务消息使用 `amqp.Persistent` |
| `MessageId` | 使用全局唯一 ID |
| `CorrelationId` | 使用请求链路 ID 或 trace ID |
| `Type` | 放事件类型 |
| `Timestamp` | 放消息创建时间或事件发生时间 |
| `Headers` | 放 schema version、tenant、retry count 等扩展信息 |

## 8. Headers 使用示例

发布：

```go
Headers: amqp.Table{
	"schema_version": int32(1),
	"source":         "order-service",
	"tenant":         "default",
}
```

消费：

```go
version, ok := d.Headers["schema_version"]
if ok {
	log.Printf("schema version: %v", version)
}
```

注意：Headers 适合放元数据，不要把大块业务数据塞进去。

## 9. 不要在消息里放什么

避免直接放：

- 密码。
- 完整身份证号。
- 银行卡号。
- 大文件内容。
- 过大的二进制数据。

大文件应该放对象存储，消息里只放引用：

```json
{
  "file_id": "file-001",
  "object_key": "uploads/2026/07/04/a.png"
}
```

## 10. 版本兼容

消息结构会变化。

例如 `order.created` v1：

```json
{
  "order_id": 1001,
  "user_id": 2001
}
```

v2 新增字段：

```json
{
  "order_id": 1001,
  "user_id": 2001,
  "amount": 9900
}
```

消费者要尽量做到：

- 新增字段不影响旧消费者。
- 删除字段要谨慎。
- 必要时通过 `version` 做分支。

## 11. 本节小结

真实项目中的 RabbitMQ 消息应该结构化。

建议：

- body 使用 JSON。
- properties 放消息元数据。
- headers 放扩展元数据。
- 每条消息有 `message_id`。
- 每条消息有 `correlation_id`。
- 每类事件有稳定的 `event_type` 和 `version`。

下一节学习消费者优雅退出和基础错误处理。

