# 4. Retry Topic 与 Dead Letter Topic

本节目标：理解 retry topic 和 dead letter topic 的作用，学会为 Go consumer 设计不会卡住主链路的失败处理流程。

Consumer 处理失败时，最糟糕的做法有两个：一是无限重试卡住 partition，二是直接提交 offset 把问题消息丢掉。Retry topic 和 DLQ 就是为了解决这个矛盾。

---

## 一、为什么需要 Retry Topic

如果处理失败就不提交 offset：

```text
consumer 会反复读到同一条消息
```

短暂错误可以这样处理。

但长期错误会导致：

- 当前 partition 卡住。
- 后续正常消息无法处理。
- lag 持续升高。
- 日志被刷屏。

Retry topic 的作用是：

```text
把暂时失败的消息转移到重试链路，让主 topic 继续前进
```

---

## 二、为什么需要 DLQ

DLQ 是 dead letter queue/topic。

它保存最终无法处理的消息。

例如：

- JSON 格式错误。
- 缺少必填字段。
- schema version 不支持。
- 重试次数耗尽。
- 业务状态无法修复。

DLQ 不是垃圾桶。

它是：

```text
需要排查、修复、可能重放的问题消息集合
```

---

## 三、Topic 命名

主 topic：

```text
order.created
```

retry topic：

```text
order.created.retry.1m
order.created.retry.5m
order.created.retry.30m
```

DLQ：

```text
order.created.dlq
```

初学项目可以先简化：

```text
order.created.retry
order.created.dlq
```

---

## 四、Retry 消息结构

```json
{
  "event_id": "evt_order_created_1001",
  "original_topic": "order.created",
  "original_partition": 1,
  "original_offset": 203,
  "original_key": "order_1001",
  "attempt": 1,
  "last_error": "database timeout",
  "failed_at": "2026-07-05T12:00:00Z",
  "payload": {}
}
```

必须包含：

- 原 topic。
- 原 partition。
- 原 offset。
- 原 key。
- attempt。
- 错误原因。
- 原始 payload。

---

## 五、DLQ 消息结构

```json
{
  "original_topic": "order.created",
  "original_partition": 1,
  "original_offset": 203,
  "original_key": "order_1001",
  "error_type": "invalid_json",
  "error_message": "unexpected end of JSON input",
  "failed_at": "2026-07-05T12:00:00Z",
  "payload": "bad-json"
}
```

DLQ 一定要保留原始 payload，否则后续无法修复和重放。

---

## 六、Offset 提交规则

| 场景 | 是否提交原 offset |
| --- | --- |
| 业务成功 | 是 |
| 写 retry topic 成功 | 是 |
| 写 DLQ 成功 | 是 |
| 写 retry topic 失败 | 否 |
| 写 DLQ 失败 | 否 |

核心原则：

```text
消息要么处理成功，要么成功转移到可靠后续链路，然后才能提交 offset
```

---

## 七、Go 伪代码

```go
err := handler.Handle(ctx, msg)
if err == nil {
    return consumer.Commit(ctx, msg)
}

if IsRetryable(err) {
    if err := retryProducer.Publish(ctx, retryTopic, BuildRetryMessage(msg, err)); err != nil {
        return err
    }
    return consumer.Commit(ctx, msg)
}

if err := dlqProducer.Publish(ctx, dlqTopic, BuildDLQMessage(msg, err)); err != nil {
    return err
}
return consumer.Commit(ctx, msg)
```

不要在写 retry/DLQ 失败时提交原 offset。

---

## 八、延迟重试怎么做

Kafka 本身不是专业延迟队列。

常见做法：

- 多级 retry topic。
- retry 消息里带 `next_retry_at`。
- retry consumer 未到时间就暂停或稍后处理。
- 使用外部调度系统。

初学项目中，多级 retry topic 足够。

---

## 九、最大重试次数

不要无限 retry。

建议：

```text
attempt >= max_attempts -> DLQ
```

例如：

```text
第 1 次失败 -> retry.1m
第 2 次失败 -> retry.5m
第 3 次失败 -> retry.30m
第 4 次失败 -> dlq
```

---

## 十、DLQ 后续处理

生产环境需要：

- DLQ 数量监控。
- 告警。
- 查询工具。
- 修复脚本。
- 重放工具。
- 人工处理记录。

DLQ 有新增不是“以后再说”，关键业务 DLQ 应立即告警。

---

## 十一、常见误区

### 1. 失败消息永远不提交 offset

坏消息会卡住主链路。

### 2. DLQ 只存错误原因

不够。

必须存原始上下文和 payload。

### 3. Retry 可以无限重试

不应该。

要有最大次数。

### 4. 写 retry 失败也提交 offset

错误。

这样消息可能丢失。

---

## 十二、本节练习

1. 为 `payment.succeeded` 设计 retry topic 和 DLQ topic。
2. 写出 retry 消息结构。
3. 写出 DLQ 消息结构。
4. 设计最大重试次数。
5. 解释为什么写 retry 成功后可以提交原 offset。
6. 解释为什么 DLQ 重放仍然需要业务幂等。

---

## 十三、本节小结

- Retry topic 用于可恢复错误。
- DLQ 用于不可处理或重试耗尽的消息。
- 写 retry/DLQ 成功后才能提交原 offset。
- Retry 消息要保留原始上下文和 attempt。
- DLQ 要保留原始 payload 和错误原因。
- Retry 必须有最大次数。
- DLQ 必须监控和告警。

---

## 十四、错误分类设计

Go consumer 里建议让 handler 返回带分类的错误：

```go
type ErrorKind string

const (
    ErrorKindRetryable    ErrorKind = "retryable"
    ErrorKindNonRetryable ErrorKind = "non_retryable"
)

type HandlerError struct {
    Kind ErrorKind
    Err  error
}

func (e HandlerError) Error() string {
    return e.Err.Error()
}

func (e HandlerError) Unwrap() error {
    return e.Err
}
```

辅助函数：

```go
func Retryable(err error) error {
    return HandlerError{Kind: ErrorKindRetryable, Err: err}
}

func NonRetryable(err error) error {
    return HandlerError{Kind: ErrorKindNonRetryable, Err: err}
}
```

这样业务 handler 不需要知道 retry topic 怎么发，只要表达错误性质。

---

## 十五、错误分类示例

```go
func (h *OrderCreatedHandler) Handle(ctx context.Context, msg Message) error {
    var event OrderCreatedEvent
    if err := json.Unmarshal(msg.Value, &event); err != nil {
        return NonRetryable(err)
    }

    if err := event.Validate(); err != nil {
        return NonRetryable(err)
    }

    if err := h.service.DeductInventory(ctx, event); err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            return Retryable(err)
        }
        return Retryable(err)
    }

    return nil
}
```

分类原则：

| 错误 | 分类 |
| --- | --- |
| JSON 解析失败 | 不可重试 |
| 必填字段为空 | 不可重试 |
| schema version 不支持 | 不可重试 |
| 数据库超时 | 可重试 |
| 数据库连接失败 | 可重试 |
| 库存不足 | 业务结果，不进 retry/DLQ |

---

## 十六、Retry Consumer 流程

retry topic 里的消息不是直接丢回主 topic 就结束。它也要判断次数：

```text
读取 retry message
如果 attempt >= max_attempts:
  写 DLQ
  提交 retry offset
否则:
  重新调用原 handler
  成功 -> 提交 retry offset
  失败 -> 写下一级 retry topic
```

伪代码：

```go
func (c *RetryConsumer) Handle(ctx context.Context, msg Message) error {
    retryMsg, err := DecodeRetryMessage(msg.Value)
    if err != nil {
        return c.publishDLQAndCommit(ctx, msg, err)
    }

    if retryMsg.Attempt >= c.maxAttempts {
        return c.publishDLQAndCommit(ctx, msg, errors.New(retryMsg.LastError))
    }

    err = c.originalHandler.Handle(ctx, Message{
        Key:   []byte(retryMsg.OriginalKey),
        Value: retryMsg.Payload,
    })
    if err == nil {
        return c.commit(ctx, msg)
    }

    return c.publishNextRetryAndCommit(ctx, msg, retryMsg, err)
}
```

---

## 十七、DLQ 重放要注意什么

DLQ 重放不是把消息复制回主 topic 那么简单。

重放前要确认：

- 失败原因已经修复。
- payload 仍然符合当前 schema。
- consumer 是幂等的。
- 重放范围明确。
- 重放后有人观察指标和日志。

重放命令可以先设计成：

```text
go run ./cmd/dlq-replay \
  --topic order.created.dlq \
  --target order.created \
  --event-id evt_order_created_order_1001
```

学习阶段可以不实现完整工具，但要知道生产里通常需要它。

---

## 十八、为什么写 retry 成功后可以提交原 offset

因为消息已经从主处理链路转移到了另一个可靠 topic。

也就是说：

```text
原消息没有处理成功。
但它已经被可靠保存到 retry topic。
主 topic 可以继续向前。
```

这和直接提交 offset 不同：

```text
直接提交 offset：消息消失。
写 retry 后提交 offset：消息换了一个地方等待处理。
```

这也是 retry topic 的核心价值。

---

## 十九、告警和运维要求

至少要观察：

```text
retry topic 消息数量
DLQ 新增数量
DLQ 最老消息时间
重试次数耗尽数量
```

告警建议：

| 条件 | 处理 |
| --- | --- |
| DLQ 5 分钟内新增 | 立即告警 |
| retry lag 持续升高 | 排查下游是否故障 |
| 同一 error_type 大量出现 | 判断是否发布了坏版本 |
| DLQ 最老消息超过 1 天 | 推进人工处理 |

---

## 二十、常见面试追问

### 1. retry topic 会不会打乱顺序？

会。进入 retry 后，失败消息和后续正常消息不再严格保持原分区顺序。设计 retry topic 本来就是在“严格顺序”和“主链路可用性”之间取舍。

### 2. 为什么不一直阻塞原 partition 等它成功？

如果是短暂错误可以等，但坏消息会让整个 partition 后续消息都无法处理。关键业务通常会选择把问题消息转移到 retry/DLQ，让主链路继续。

### 3. DLQ 消息怎么处理？

先告警和定位，再修复数据或代码，最后通过受控工具重放。DLQ 不应该没人管。
