# 4. 封装 Consumer Group

本节目标：使用 Go 封装 Kafka consumer group，支持 handler、手动提交 offset、错误返回和优雅退出的基础结构。

Consumer 比 producer 更容易出问题。Producer 发送失败通常比较直接，而 consumer 涉及业务处理、offset 提交、重复消费、rebalance 和退出。封装 consumer 时，重点是把这些控制点留出来。

---

## 一、定义 Handler 接口

`internal/kafka/consumer.go`：

```go
package kafka

import "context"

type Handler interface {
    Handle(ctx context.Context, msg ConsumedMessage) error
}

type ConsumedMessage struct {
    Topic     string
    Partition int32
    Offset    int64
    Key       []byte
    Value     []byte
    Headers   map[string]string
}
```

业务模块实现 `Handler`。

Kafka 封装只负责：

- 拉消息。
- 调用 handler。
- 根据结果提交 offset。
- 记录日志。

---

## 二、Consumer 配置

```go
type ConsumerConfig struct {
    Brokers []string
    ClientID string
    GroupID string
    Topics []string
}
```

环境变量示例：

```text
KAFKA_BROKERS=localhost:9092
KAFKA_CLIENT_ID=inventory-service
KAFKA_GROUP_ID=inventory-service
KAFKA_TOPICS=order.created
```

---

## 三、创建 Franz Consumer

```go
type FranzConsumer struct {
    client  *kgo.Client
    handler Handler
}

func NewFranzConsumer(cfg ConsumerConfig, handler Handler) (*FranzConsumer, error) {
    client, err := kgo.NewClient(
        kgo.SeedBrokers(cfg.Brokers...),
        kgo.ClientID(cfg.ClientID),
        kgo.ConsumerGroup(cfg.GroupID),
        kgo.ConsumeTopics(cfg.Topics...),
        kgo.DisableAutoCommit(),
    )
    if err != nil {
        return nil, err
    }

    return &FranzConsumer{
        client:  client,
        handler: handler,
    }, nil
}
```

关键配置：

```go
kgo.DisableAutoCommit()
```

表示关闭自动提交，改为业务成功后手动提交。

---

## 四、Run 主循环

```go
func (c *FranzConsumer) Run(ctx context.Context) error {
    for ctx.Err() == nil {
        fetches := c.client.PollFetches(ctx)

        if errs := fetches.Errors(); len(errs) > 0 {
            for _, fetchErr := range errs {
                // 生产代码中记录 topic、partition、错误
                _ = fetchErr
            }
            continue
        }

        fetches.EachRecord(func(record *kgo.Record) {
            msg := toConsumedMessage(record)

            if err := c.handler.Handle(ctx, msg); err != nil {
                // 后续章节会接 retry / DLQ
                return
            }

            _ = c.client.CommitRecords(ctx, record)
        })
    }

    return ctx.Err()
}
```

这只是基础版。

生产版不能忽略 handler 错误，也不能忽略 commit 错误。

---

## 五、Record 转换

```go
func toConsumedMessage(record *kgo.Record) ConsumedMessage {
    headers := make(map[string]string, len(record.Headers))
    for _, h := range record.Headers {
        headers[h.Key] = string(h.Value)
    }

    return ConsumedMessage{
        Topic:     record.Topic,
        Partition: record.Partition,
        Offset:    record.Offset,
        Key:       record.Key,
        Value:     record.Value,
        Headers:   headers,
    }
}
```

业务 handler 不需要依赖 `kgo.Record`。

这是封装的意义。

---

## 六、业务 Handler 示例

`internal/inventory/handler.go`：

```go
type OrderCreatedHandler struct {
    service *Service
}

func (h *OrderCreatedHandler) Handle(ctx context.Context, msg kafka.ConsumedMessage) error {
    var event OrderCreatedEvent
    if err := json.Unmarshal(msg.Value, &event); err != nil {
        return err
    }

    return h.service.Deduct(ctx, event)
}
```

后续会改进：

- JSON 解析错误进入 DLQ。
- 数据库错误进入 retry。
- 业务幂等使用 `processed_events`。

---

## 七、Commit 错误处理

基础示例里忽略了 commit 错误，生产中不能这样。

应该记录：

```text
topic
partition
offset
key
event_id
error
```

commit 失败意味着：

```text
消息可能会重复消费
```

所以业务 handler 必须幂等。

---

## 八、Close 方法

```go
func (c *FranzConsumer) Close(ctx context.Context) error {
    c.client.Close()
    return nil
}
```

服务退出时要：

- 取消 context。
- 停止拉新消息。
- 等当前处理完成。
- close consumer。

完整优雅退出会在后续实践中展开。

---

## 九、当前版本还缺什么

这个基础版还缺：

- retry topic。
- DLQ。
- handler timeout。
- metrics。
- commit 错误日志。
- rebalance 日志。
- 幂等。
- 优雅退出等待。

但它已经体现了最重要的骨架：

```text
拉消息 -> handler -> 成功后 commit
```

---

## 十、常见误区

### 1. Handler 直接提交 offset

不推荐。

offset 提交属于 Kafka 封装层职责。

### 2. Kafka 封装层写业务逻辑

不推荐。

Kafka 层不应该知道库存、订单、支付。

### 3. 自动提交更省事

关键业务不推荐。

### 4. commit 错误可以忽略

不推荐。

至少要记录并依赖幂等兜底。

---

## 十一、本节练习

1. 定义 `Handler` 接口。
2. 定义 `ConsumedMessage`。
3. 创建 `FranzConsumer`。
4. 关闭自动提交。
5. handler 成功后提交 offset。
6. 写一个 `OrderCreatedHandler` 解码消息。

---

## 十二、本节小结

- Consumer 封装要支持 handler。
- Kafka 层负责拉取和提交 offset。
- 业务层负责处理消息。
- 关键业务关闭自动提交。
- handler 成功后再 commit。
- commit 失败可能导致重复消费。
- 业务 handler 必须幂等。

---

## 十四、Handler 接口

建议定义：

```go
type Handler interface {
    Handle(ctx context.Context, msg Message) error
}
```

Kafka 层只负责拉消息、调用 handler、处理 retry/DLQ、提交 offset。

业务层负责解析事件、校验字段、执行业务事务、返回错误分类。

---

## 十五、Run 方法验收

`Run(ctx)` 至少要做到：

- `ctx` 取消时能退出。
- handler 成功后提交 offset。
- handler 失败时不静默提交。
- commit 失败记录日志。
- 退出时关闭客户端。

---

## 十六、Run 伪代码

```go
func (c *ConsumerGroup) Run(ctx context.Context, handler Handler) error {
    for ctx.Err() == nil {
        msg, err := c.fetch(ctx)
        if err != nil {
            return err
        }

        if err := handler.Handle(ctx, msg); err != nil {
            return err
        }

        if err := c.commit(ctx, msg); err != nil {
            c.logger.Error("commit failed", "error", err)
        }
    }
    return ctx.Err()
}
```

这是最小骨架。后续再在 handler error 分支中加入 retry/DLQ。

---

## 十七、测试建议

用 fake handler 测试：

- handler 返回 nil 时会调用 commit。
- handler 返回 error 时不会提交。
- context 取消时能退出。
- commit 返回 error 时会记录日志。

Consumer 封装必须先把这些边界测清楚。

---

## 十八、测试用例模板

```go
func TestConsumerCommitsAfterSuccess(t *testing.T) {
    handler := HandlerFunc(func(ctx context.Context, msg Message) error {
        return nil
    })

    consumer := NewFakeConsumer()
    err := consumer.RunOnce(context.Background(), handler)

    require.NoError(t, err)
    require.True(t, consumer.Committed)
}
```

真实项目可以先用 fake consumer 验证提交边界，再做集成测试。

---

## 二十一、封装后的测试重点

ConsumerGroup 封装完成后，至少写下面几类测试：

```text
handler 成功时是否提交 offset。
handler 返回可重试错误时是否不提交。
handler 返回不可重试错误时是否进入 DLQ。
context 取消时是否停止拉取。
关闭时是否等待正在处理的消息结束。
日志是否包含 event_id、topic、partition、offset。
```

这些测试不一定都需要真实 Kafka。很多边界可以先用 fake message、fake committer 和 fake DLQ publisher 验证。
