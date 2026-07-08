# 10. RabbitMQ Go 项目封装建议

## 1. 为什么要封装

第 3、4 阶段的 demo 中，很多代码都写在 `main.go` 里。

真实项目不能这样散落：

- 连接管理重复。
- 发布逻辑重复。
- confirm 处理重复。
- JSON 编解码重复。
- consumer ack/retry/DLQ 逻辑重复。
- 日志和指标不统一。

所以需要适度封装。

注意：

```text
不要一开始就写一个巨大 MQ SDK。
```

先让业务跑通，再抽取稳定重复的部分。

## 2. 推荐项目结构

```text
internal/
  mq/
    config.go
    connection.go
    topology.go
    publisher.go
    consumer.go
    message.go
    retry.go
  outbox/
    message.go
    repository.go
    worker.go
  app/
    order/
      service.go
      events.go
      handlers.go
    notification/
      service.go
      handlers.go
```

职责：

```text
mq       -> RabbitMQ 通用能力
outbox   -> Outbox 模式
app      -> 业务逻辑
```

## 3. 配置结构

```go
type Config struct {
	URL             string
	PublishTimeout  time.Duration
	ConfirmTimeout  time.Duration
	PrefetchCount   int
	ConsumerTag     string
}
```

不要硬编码：

```go
const rabbitURL = "amqp://..."
```

生产环境应该从：

- 环境变量。
- 配置文件。
- 配置中心。
- Kubernetes Secret。

读取。

## 4. Publisher 接口

```go
type Publisher interface {
	Publish(ctx context.Context, msg PublishMessage) error
}

type PublishMessage struct {
	Exchange      string
	RoutingKey    string
	MessageID     string
	CorrelationID string
	EventType     string
	Body          []byte
	Headers       amqp.Table
}
```

实现时包含：

- `PublishWithContext`
- `DeliveryMode: amqp.Persistent`
- publisher confirm
- mandatory publish
- return 处理
- 统一日志

## 5. Consumer Handler 接口

```go
type Handler interface {
	Handle(ctx context.Context, msg Message) error
}

type HandlerFunc func(ctx context.Context, msg Message) error

func (f HandlerFunc) Handle(ctx context.Context, msg Message) error {
	return f(ctx, msg)
}
```

消息结构：

```go
type Message struct {
	MessageID     string
	CorrelationID string
	EventType     string
	RoutingKey    string
	Headers       amqp.Table
	Body          []byte
}
```

## 6. Consumer 封装要做什么

Consumer 通用逻辑：

```text
连接 RabbitMQ
声明 topology
设置 prefetch
注册 consumer
收到 delivery
转成 Message
调用业务 Handler
成功 -> ack
失败 -> retry 或 DLQ
```

业务 Handler 只关心：

```text
如何处理这条业务消息。
```

不应该每个业务 handler 都重复写 ack/nack/retry。

## 7. 错误类型

建议定义错误类型：

```go
var (
	ErrBadMessage     = errors.New("bad message")
	ErrDuplicate      = errors.New("duplicate message")
	ErrRetryable      = errors.New("retryable error")
	ErrNonRetryable   = errors.New("non-retryable error")
)
```

或者使用自定义类型：

```go
type HandlerError struct {
	Retryable bool
	Cause     error
}
```

Consumer 根据错误类型决定：

```text
bad message -> DLQ
duplicate -> ack
retryable -> retry queue
non-retryable -> DLQ
success -> ack
```

## 8. Topology 声明

不要把 exchange、queue、binding 散落在各处。

可以定义：

```go
type ExchangeConfig struct {
	Name string
	Kind string
}

type QueueConfig struct {
	Name string
	Args amqp.Table
}

type BindingConfig struct {
	Queue      string
	Exchange   string
	RoutingKey string
}
```

启动时统一声明。

## 9. 日志字段

每次发布和消费都记录：

```text
message_id
correlation_id
event_type
exchange
routing_key
queue
retry_count
duration
error
```

不要只写：

```text
handle failed
```

要能排查是哪条消息失败。

## 10. 封装边界

不要把业务逻辑塞进 `mq` 包。

`mq` 包只处理：

- RabbitMQ 连接。
- 发布。
- 消费。
- ack/retry/DLQ。
- 编解码辅助。

业务逻辑放在：

```text
internal/app/order
internal/app/notification
```

## 11. 本节小结

RabbitMQ 封装建议：

- 小步封装，不要一开始做大而全 SDK。
- Publisher 统一处理 confirm 和 mandatory。
- Consumer 统一处理 ack、retry、DLQ。
- Handler 只写业务逻辑。
- 日志字段统一。
- topology 集中声明。

下一节学习集成测试和本地演练。

