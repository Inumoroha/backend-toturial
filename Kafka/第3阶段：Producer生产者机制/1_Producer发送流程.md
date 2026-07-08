# 1. Producer 发送流程

本节目标：理解 Kafka producer 从业务代码调用发送，到 broker 返回确认，中间经历了哪些步骤，以及这些步骤如何影响 Go 后端的错误处理、日志和超时控制。

很多初学者以为 producer 就是：

```text
调用 send 方法，消息就进 Kafka
```

真实情况更复杂。Producer 内部会序列化消息、选择 partition、进入缓冲区、组成 batch、发送到 broker、等待确认、处理重试和回调。

---

## 一、Producer 在 Go 后端中的位置

以订单服务为例：

```text
HTTP Handler
  -> Order Service
  -> 写 PostgreSQL
  -> 发布 order.created 到 Kafka
```

Producer 不是独立业务系统，它通常被封装在 Go 服务内部。

常见代码位置：

```text
internal/kafka/producer.go
internal/order/service.go
```

业务层不应该直接散落调用第三方 Kafka 客户端，而应该调用你自己封装的接口：

```go
type Producer interface {
    Publish(ctx context.Context, topic string, msg Message) error
}
```

---

## 二、一条消息发送时发生了什么

简化流程：

```text
业务代码构造事件
序列化为 JSON/Protobuf
设置 topic、key、headers
producer 选择 partition
消息进入 producer buffer
多个消息组成 batch
发送请求到 partition leader
broker 写入日志
broker 返回 ack
producer 返回成功或失败
```

每一步都可能影响可靠性和性能。

---

## 三、消息结构

一条 Kafka record 通常包含：

- topic
- key
- value
- headers
- timestamp

业务上建议 value 是统一事件结构：

```json
{
  "event_id": "evt_order_created_1001",
  "event_type": "order.created",
  "version": 1,
  "occurred_at": "2026-07-05T12:00:00Z",
  "data": {
    "order_id": "order_1001",
    "user_id": "user_88",
    "total_amount": 19800
  }
}
```

headers 可以放：

```text
trace_id
event_id
event_type
schema_version
source_service
```

headers 不应该承载大块业务数据。业务数据放 value。

---

## 四、序列化

Producer 发送前需要把 Go struct 转成字节。

学习阶段可以用 JSON：

```go
payload, err := json.Marshal(event)
if err != nil {
    return err
}
```

生产环境也可能用：

- JSON
- Protobuf
- Avro

JSON 优点是直观、便于命令行查看。缺点是体积较大、类型约束弱。

---

## 五、选择 Partition

如果 record 有 key，producer 通常根据 key 选择 partition。

例如：

```go
Key: []byte(orderID)
```

这让同一个订单的事件进入同一个 partition，从而获得单订单维度的顺序。

如果没有 key，客户端可能轮询或粘性分区，具体取决于客户端实现。

Go 后端关键业务建议显式设置 key，不要依赖默认行为。

---

## 六、Buffer 与 Batch

Producer 不一定每条消息都立刻发一个网络请求。

为了提高吞吐，它通常会：

```text
把消息放入本地 buffer
按 topic-partition 聚合
形成 batch
再发送给 broker
```

这就是为什么有 `batch.size`、`linger.ms` 之类参数。

直觉：

- batch 大：吞吐可能更高。
- linger 大：更容易凑 batch，但延迟可能增加。
- buffer 满：发送可能阻塞或失败。

---

## 七、Broker 写入与 Ack

Producer 请求最终会发送到目标 partition 的 leader broker。

broker 收到后会：

```text
追加写入 partition log
根据 acks 配置等待确认条件
返回成功或失败
```

`acks` 后续会详细讲。

这里先记住：

```text
producer 返回成功，不只是本地函数执行完，而是 broker 按配置完成了确认
```

---

## 八、同步发送与异步发送

### 同步发送

业务等待发送结果：

```text
Publish(ctx, event) -> 等待成功或失败
```

适合：

- 关键业务事件。
- 请求成功需要确保事件已写入 Kafka。

缺点：

- 增加请求延迟。

### 异步发送

调用后先返回，通过回调或 delivery report 得到结果。

适合：

- 行为日志。
- 埋点。
- 高吞吐场景。

注意：

```text
异步发送不是不管结果
```

仍然要处理发送失败。

---

## 九、Context 超时

Go producer 必须支持 context。

示例：

```go
ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
defer cancel()

err := producer.Publish(ctx, "order.created", msg)
```

为什么？

- broker 可能不可用。
- 网络可能抖动。
- buffer 可能满。
- 业务请求不能无限等待。

超时后要记录日志：

```text
topic=order.created key=order_1001 event_id=evt_1 error=timeout
```

---

## 十、Producer 日志字段

发送成功日志：

```text
level=info msg="kafka publish success" topic=order.created key=order_1001 event_id=evt_1 duration_ms=15
```

发送失败日志：

```text
level=error msg="kafka publish failed" topic=order.created key=order_1001 event_id=evt_1 error="request timeout"
```

至少记录：

- topic
- key
- event_id
- event_type
- duration_ms
- error

---

## 十一、Go 封装示例

接口：

```go
type Message struct {
    Key     []byte
    Value   []byte
    Headers map[string]string
}

type Producer interface {
    Publish(ctx context.Context, topic string, msg Message) error
    Close(ctx context.Context) error
}
```

业务代码只依赖这个接口：

```go
err := p.Publish(ctx, "order.created", kafka.Message{
    Key:   []byte(order.ID),
    Value: payload,
    Headers: map[string]string{
        "event_id": event.EventID,
    },
})
```

这样业务层不需要知道底层使用 `franz-go`、`sarama` 还是 `confluent-kafka-go`。

---

## 十二、常见错误

### 1. 每条消息创建一个 producer

错误。

Producer 应该长期复用。

每条消息都创建 producer 会导致：

- 连接频繁创建。
- 性能差。
- 资源泄漏。
- 难以 flush。

### 2. 发送失败只打印日志

关键业务不能只打印日志。

需要考虑：

- 返回错误。
- 写 outbox。
- 重试。
- 告警。

### 3. 不设置 key

如果业务依赖顺序，不设置 key 可能导致同一业务对象进入不同 partition。

### 4. 退出时不 close

异步发送场景下，不 close 或 flush 可能导致缓冲区消息丢失。

---

## 十三、本节练习

1. 画出 producer 发送一条消息的完整流程。
2. 为 `order.created` 设计 key、value、headers。
3. 说明同步发送和异步发送分别适合什么场景。
4. 设计一个 `Producer` 接口。
5. 思考：订单写库成功但 Kafka 发送失败，应该怎么办？

---

## 十四、本节小结

- Producer 发送不是简单函数调用，中间包含序列化、分区、缓冲、批量、网络请求和 ack。
- Go 后端中 producer 应统一封装。
- 关键业务消息要设置稳定 key。
- value 承载业务事件，headers 承载元数据。
- Producer 应支持 context timeout。
- 异步发送也必须处理结果。
- 退出时要 close 或 flush producer。
- 发送失败处理要和业务可靠性要求匹配。

---

## 十五、发送失败排查顺序

如果 producer 发送失败，按顺序查：

1. `bootstrap.servers` 是否正确。
2. topic 是否存在。
3. message 是否超过大小限制。
4. key/value 序列化是否失败。
5. broker 是否可用。
6. acks 等待是否超时。
7. ACL 是否有写权限。

不要只看一条 `send failed` 日志。Producer 发送链路很长，错误可能发生在客户端、网络、broker 或权限层。

---

## 十六、日志字段建议

Producer 发送日志至少包含：

```text
topic
key
event_id
event_type
duration_ms
error
```

如果客户端能返回 partition 和 offset，也一起记录。这样后续 consumer 侧排查时可以串起完整链路。

---

## 二十一、发送链路复盘问题

每次调试 Producer，都可以问自己：

```text
消息是在业务事务前发送，还是事务后发送？
发送成功是否拿到了 topic、partition、offset？
发送失败是可重试错误，还是不可重试错误？
调用方是否会因为超时重复调用发送逻辑？
日志里是否能根据 event_id 找到同一条消息的所有记录？
```

这些问题可以帮助你把“调用 Send 方法”提升成“设计一次可靠的事件发布”。

后面学习 Outbox 时，你会发现很多发送链路问题本质上都和业务事务边界有关。
