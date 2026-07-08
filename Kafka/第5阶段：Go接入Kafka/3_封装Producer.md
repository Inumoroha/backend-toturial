# 3. 封装 Producer

本节目标：使用 `franz-go` 封装一个 Go Kafka producer，支持 context 超时、headers、统一日志和关闭逻辑。

本节不是为了写一个完美的生产库，而是搭出清晰骨架：业务层如何调用，Kafka 层如何隐藏客户端细节，错误如何返回。

---

## 一、安装依赖

```powershell
go get github.com/twmb/franz-go/pkg/kgo
```

---

## 二、定义接口

`internal/kafka/producer.go`：

```go
package kafka

import "context"

type Producer interface {
    Publish(ctx context.Context, topic string, msg Message) error
    Close(ctx context.Context) error
}
```

`internal/kafka/message.go`：

```go
package kafka

type Message struct {
    Key     []byte
    Value   []byte
    Headers map[string]string
}
```

---

## 三、实现结构体

```go
package kafka

import (
    "context"
    "fmt"

    "github.com/twmb/franz-go/pkg/kgo"
)

type FranzProducer struct {
    client *kgo.Client
}

func NewFranzProducer(cfg Config) (*FranzProducer, error) {
    client, err := kgo.NewClient(
        kgo.SeedBrokers(cfg.Brokers...),
        kgo.ClientID(cfg.ClientID),
        kgo.RequiredAcks(kgo.AllISRAcks()),
    )
    if err != nil {
        return nil, err
    }

    return &FranzProducer{client: client}, nil
}
```

这里使用：

```go
kgo.RequiredAcks(kgo.AllISRAcks())
```

表示按 ISR 确认，适合关键业务学习示例。

---

## 四、转换 Headers

```go
func toRecordHeaders(headers map[string]string) []kgo.RecordHeader {
    result := make([]kgo.RecordHeader, 0, len(headers))
    for k, v := range headers {
        result = append(result, kgo.RecordHeader{
            Key:   k,
            Value: []byte(v),
        })
    }
    return result
}
```

headers 用于放元数据：

```text
event_id
event_type
trace_id
schema_version
```

---

## 五、Publish 方法

```go
func (p *FranzProducer) Publish(ctx context.Context, topic string, msg Message) error {
    record := &kgo.Record{
        Topic:   topic,
        Key:     msg.Key,
        Value:   msg.Value,
        Headers: toRecordHeaders(msg.Headers),
    }

    results := p.client.ProduceSync(ctx, record)
    if err := results.FirstErr(); err != nil {
        return fmt.Errorf("publish kafka message topic=%s key=%s: %w",
            topic, string(msg.Key), err)
    }

    return nil
}
```

这里使用同步发送，适合关键业务入门。

后续如果做高吞吐日志，可以改成异步发送，但必须处理 delivery report。

---

## 六、Close 方法

```go
func (p *FranzProducer) Close(ctx context.Context) error {
    p.client.Close()
    return nil
}
```

不同客户端 close/flush 语义不同。你封装接口的好处是：业务层不需要知道底层细节。

---

## 七、业务层调用

`internal/order/service.go`：

```go
func (s *Service) PublishOrderCreated(ctx context.Context, event OrderCreatedEvent) error {
    payload, err := json.Marshal(event)
    if err != nil {
        return err
    }

    msg := kafka.Message{
        Key:   []byte(event.Data.OrderID),
        Value: payload,
        Headers: map[string]string{
            "event_id":       event.EventID,
            "event_type":     event.EventType,
            "schema_version": strconv.Itoa(event.Version),
        },
    }

    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()

    return s.producer.Publish(ctx, "order.created", msg)
}
```

注意：

- key 使用 `order_id`。
- value 是完整事件 JSON。
- headers 放元数据。
- 使用 context timeout。

---

## 八、日志应该放在哪里

可以在 `Publish` 中记录基础日志，也可以由业务层记录。

推荐 producer 封装至少记录失败：

```text
topic
key
event_id
error
```

业务层可以记录：

```text
order_id
user_id
```

不要吞掉错误。

---

## 九、发送失败如何处理

关键业务不要只打印日志。

可选策略：

1. 返回错误，让请求失败。
2. 使用 outbox，后台重试发送。
3. 本地短重试，但要有上限。

订单类项目更推荐 outbox。

---

## 十、常见误区

### 1. producer 每次 Publish 都 NewClient

错误。

producer 应该复用。

### 2. Publish 没有 context

不推荐。

发送可能阻塞或超时。

### 3. 不设置 key

订单事件不推荐。

### 4. 发送失败被忽略

关键业务不允许。

---

## 十一、本节练习

1. 创建 `internal/kafka/producer.go`。
2. 实现 `NewFranzProducer`。
3. 实现 `Publish`。
4. 定义 `OrderCreatedEvent`。
5. 调用 producer 发布 `order.created`。
6. 使用命令行 consumer 验证消息。

---

## 十二、本节小结

- Producer 应该封装在 `internal/kafka`。
- 业务层依赖自定义接口。
- `Publish` 应支持 context。
- 关键业务使用稳定 key。
- headers 放 event_id、event_type 等元数据。
- 发送失败必须返回或进入补偿流程。

---

## 十三、Producer 接口建议

业务层最好依赖一个小接口：

```go
type Producer interface {
    Publish(ctx context.Context, topic string, msg Message) error
}

type Message struct {
    Key     []byte
    Value   []byte
    Headers map[string]string
}
```

这样 `order-service` 不需要知道底层用的是 `franz-go`、`segmentio/kafka-go` 还是其他客户端。

---

## 十四、Publish 必须做的事

`Publish` 至少要处理：

- context 超时。
- topic 不能为空。
- key/value 传入。
- headers 转换。
- 客户端发送错误返回。
- 关闭时不再接收新消息。

不要在封装里吞错误。吞错会让业务层误以为消息已经发送成功。

---

## 十五、测试建议

Producer 封装至少写两类测试：

```text
成功发送时调用底层 client。
底层 client 返回错误时，Publish 返回错误。
```

关键链路还要用集成测试验证消息真的进入 Kafka。

---

## 十六、关闭 Producer

服务退出时要关闭 producer：

```go
defer producer.Close()
```

如果底层客户端有 flush 能力，退出前要等待缓冲消息发送完成，避免进程退出导致消息留在本地内存。

---

## 十七、Publish 调用示例

业务层调用时应该像这样：

```go
msg := kafka.Message{
    Key:   []byte(orderID),
    Value: payload,
    Headers: map[string]string{
        "event_id":   eventID,
        "event_type": "order.created",
    },
}

if err := producer.Publish(ctx, "order.created", msg); err != nil {
    return fmt.Errorf("publish order.created: %w", err)
}
```

重点是错误要继续向上返回。关键业务不要只打印日志。

---

## 十八、封装验收清单

- [ ] 支持传入 topic。
- [ ] 支持 message key。
- [ ] 支持 headers。
- [ ] 支持 context timeout。
- [ ] 底层发送失败会返回 error。
- [ ] close/flush 有明确调用位置。
- [ ] 可以用 fake producer 测试业务 service。

满足这些，Producer 才算是工程封装，而不是简单 demo。

---

## 十九、Fake Producer 示例

业务单元测试可以使用 fake：

```go
type FakeProducer struct {
    Messages []kafka.Message
    Err      error
}

func (f *FakeProducer) Publish(ctx context.Context, topic string, msg kafka.Message) error {
    if f.Err != nil {
        return f.Err
    }
    f.Messages = append(f.Messages, msg)
    return nil
}
```

这样可以测试业务是否正确构造 key、headers 和 payload，而不依赖真实 Kafka。

---

## 二十、最终练习

为 `CreateOrderService` 写一个单元测试：

```text
调用 CreateOrder。
断言 fake producer 收到 1 条消息。
断言 key 等于 order_id。
断言 header 中有 event_id。
```

这能证明 producer 封装确实服务于业务测试。

---

## 二十、Producer 接口不要泄漏太多细节

业务层更适合依赖下面这种接口：

```go
type EventPublisher interface {
    PublishOrderCreated(ctx context.Context, event OrderCreatedEvent) error
}
```

而不是到处传 Kafka message、partition、header 等底层结构。

底层细节可以留在 adapter 中处理。这样做的好处是：业务测试可以替换 fake publisher，未来换 Kafka 客户端库时，业务代码也不需要大面积修改。
