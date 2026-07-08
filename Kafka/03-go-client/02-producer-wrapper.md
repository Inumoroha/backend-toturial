# 02 封装 Go Producer

本节目标：设计一个 Go 项目中可维护的 Kafka producer 封装。

## 为什么要封装

不要在业务代码里直接到处调用 Kafka 客户端 API。

原因：

- 配置会散落在各处。
- 日志字段不统一。
- 发送失败处理不统一。
- 后续切换客户端困难。
- 测试时不好 mock。

## Producer 封装目标

一个好的 producer wrapper 应该支持：

- 统一配置。
- `context.Context` 超时。
- 结构化消息。
- 自动填充 event id 和 timestamp。
- 同步发送或等待发送结果。
- 统一日志。
- 发送失败返回明确错误。
- 服务退出时 close 或 flush。

## 推荐接口

```go
type Event struct {
    ID        string
    Type      string
    Version   int
    Key       string
    Payload   []byte
    Headers   map[string]string
    OccurredAt time.Time
}

type Producer interface {
    Publish(ctx context.Context, topic string, event Event) error
    Close(ctx context.Context) error
}
```

## 事件字段建议

每个事件至少包含：

- `event_id`
- `event_type`
- `version`
- `occurred_at`
- `data`

headers 建议包含：

- `trace_id`
- `source_service`
- `schema_version`
- `content_type`

## 发送超时

不要让 producer 无限等待。

业务层可以这样使用：

```go
ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
defer cancel()

err := producer.Publish(ctx, "order.created", event)
```

超时时间要根据业务可靠性和请求链路设计。

## 日志建议

发送成功：

```text
level=info msg="kafka publish success" topic=order.created key=order_1001 event_id=evt_1 duration_ms=12
```

发送失败：

```text
level=error msg="kafka publish failed" topic=order.created key=order_1001 event_id=evt_1 error="timeout"
```

## 同步发送和异步发送

### 同步发送

业务等待发送结果。

适合：

- 关键业务事件。
- 请求成功必须确保事件已经写入 Kafka。

### 异步发送

业务不阻塞等待每条消息结果，但需要后台处理 delivery report。

适合：

- 日志采集。
- 行为埋点。
- 吞吐优先场景。

注意：异步发送不是“不管结果”。失败仍然要记录和处理。

## 本节练习

1. 写出你项目里的 `Producer` 接口。
2. 设计 `OrderCreatedEvent` 结构体。
3. 思考：创建订单接口应该同步等待 Kafka 发送成功吗？
4. 思考：如果数据库写入成功但 Kafka 发送失败，应该怎么办？

