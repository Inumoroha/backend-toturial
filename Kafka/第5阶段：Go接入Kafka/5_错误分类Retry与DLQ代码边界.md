# 5. 错误分类、Retry 与 DLQ 代码边界

本节目标：在 Go consumer 封装中设计错误分类、retry topic 和 DLQ 的代码边界，明确哪些逻辑属于 Kafka 层，哪些逻辑属于业务 handler。

Consumer 的质量不取决于正常消息能不能处理，而取决于异常消息如何处理。真实项目中，JSON 解析失败、数据库超时、下游服务不可用、业务状态异常都会发生。

---

## 一、为什么要错误分类

不是所有错误都应该同样处理。

例如：

```text
数据库临时超时 -> 可以重试
JSON 格式错误 -> 重试没有意义
缺少 order_id -> 重试没有意义
下游库存服务限流 -> 可以稍后重试
```

如果所有错误都不提交 offset：

```text
坏消息会一直卡住 partition
```

如果所有错误都提交 offset：

```text
可恢复错误可能被直接跳过
```

---

## 二、定义错误类型

可以定义：

```go
package kafka

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

---

## 三、判断错误类型

```go
func Classify(err error) ErrorKind {
    var handlerErr HandlerError
    if errors.As(err, &handlerErr) {
        return handlerErr.Kind
    }

    return ErrorKindRetryable
}
```

默认把未知错误当成可重试，更保守。

但生产中要结合业务：

- 数据校验错误通常不可重试。
- 系统临时错误通常可重试。

---

## 四、Handler 如何返回错误

```go
func (h *OrderCreatedHandler) Handle(ctx context.Context, msg kafka.ConsumedMessage) error {
    var event OrderCreatedEvent
    if err := json.Unmarshal(msg.Value, &event); err != nil {
        return kafka.NonRetryable(err)
    }

    if event.Data.OrderID == "" {
        return kafka.NonRetryable(errors.New("missing order_id"))
    }

    if err := h.service.Deduct(ctx, event); err != nil {
        return kafka.Retryable(err)
    }

    return nil
}
```

业务 handler 不直接写 retry topic 或 DLQ。

它只告诉 Kafka 封装层：

```text
这个错误能不能重试
```

---

## 五、Retry Producer 与 DLQ Producer

Consumer 封装可以持有两个 producer：

```go
type FranzConsumer struct {
    client        *kgo.Client
    handler       Handler
    retryProducer Producer
    dlqProducer   Producer
}
```

主流程：

```text
handler 成功 -> commit
retryable error -> publish retry -> commit
non-retryable error -> publish dlq -> commit
```

如果 publish retry/DLQ 失败：

```text
不提交原 offset
```

---

## 六、Retry 消息结构

```go
type RetryMessage struct {
    OriginalTopic     string          `json:"original_topic"`
    OriginalPartition int32           `json:"original_partition"`
    OriginalOffset    int64           `json:"original_offset"`
    OriginalKey       string          `json:"original_key"`
    Attempt           int             `json:"attempt"`
    ErrorMessage      string          `json:"error_message"`
    Payload           json.RawMessage `json:"payload"`
    FailedAt          time.Time       `json:"failed_at"`
}
```

必须保留原始上下文。

---

## 七、DLQ 消息结构

```go
type DLQMessage struct {
    OriginalTopic     string          `json:"original_topic"`
    OriginalPartition int32           `json:"original_partition"`
    OriginalOffset    int64           `json:"original_offset"`
    OriginalKey       string          `json:"original_key"`
    ErrorKind         string          `json:"error_kind"`
    ErrorMessage      string          `json:"error_message"`
    Payload           json.RawMessage `json:"payload"`
    FailedAt          time.Time       `json:"failed_at"`
}
```

DLQ 的目标不是“丢掉坏消息”，而是保留足够信息方便修复和重放。

---

## 八、Consumer 错误处理伪代码

```go
func (c *FranzConsumer) handleError(ctx context.Context, msg ConsumedMessage, record *kgo.Record, err error) {
    switch Classify(err) {
    case ErrorKindRetryable:
        if c.publishRetry(ctx, msg, err) == nil {
            _ = c.client.CommitRecords(ctx, record)
        }
    case ErrorKindNonRetryable:
        if c.publishDLQ(ctx, msg, err) == nil {
            _ = c.client.CommitRecords(ctx, record)
        }
    }
}
```

生产代码中不能忽略 publish 和 commit 错误，要记录日志和指标。

---

## 九、Topic 命名

主 topic：

```text
order.created
```

retry topic：

```text
order.created.retry.1m
order.created.retry.5m
```

DLQ：

```text
order.created.dlq
```

初学项目可以先只实现一层 retry：

```text
order.created.retry
```

---

## 十、常见误区

### 1. Handler 自己提交 offset

不推荐。

offset 提交策略应该统一在 Kafka consumer 封装里。

### 2. Handler 自己写 DLQ

也不推荐作为默认设计。

业务 handler 返回错误类型，Kafka 层统一写 DLQ，更一致。

### 3. 写 retry 失败也提交 offset

不应该。

这样原消息可能既没处理，也没进入后续链路。

### 4. DLQ 不保存原始 payload

不应该。

没有原始 payload 就难以修复重放。

---

## 十一、本节练习

1. 定义 `HandlerError`。
2. 实现 `Retryable` 和 `NonRetryable`。
3. 为 JSON 解析失败返回不可重试错误。
4. 为数据库超时返回可重试错误。
5. 设计 retry 消息结构。
6. 设计 DLQ 消息结构。
7. 写出 handler 错误处理伪代码。

---

## 十二、本节小结

- Consumer 错误必须分类。
- 可重试错误进入 retry topic。
- 不可重试错误进入 DLQ。
- 写 retry/DLQ 成功后才能提交原 offset。
- Handler 不直接提交 offset。
- DLQ 必须保存原始上下文和 payload。
- 错误分类是可靠消费的代码基础。

---

## 十四、代码边界再强调

handler 只表达结果：

```go
return kafka.Retryable(err)
return kafka.NonRetryable(err)
return nil
```

consumer 框架负责动作：

```text
retryable -> publish retry -> commit
non_retryable -> publish DLQ -> commit
nil -> commit
```

不要让 handler 自己写 retry topic 或提交 offset，否则每个业务 handler 都会复制一套可靠性代码。

---

## 十五、错误类型示例

| 错误 | handler 返回 |
| --- | --- |
| JSON 解析失败 | `NonRetryable` |
| 字段校验失败 | `NonRetryable` |
| 数据库超时 | `Retryable` |
| Kafka retry topic 写失败 | consumer 框架处理 |
| 库存不足 | 业务失败事件 |

---

## 十六、Consumer 框架处理模板

```go
err := handler.Handle(ctx, msg)
if err == nil {
    return commit(ctx, msg)
}

switch Classify(err) {
case ErrorKindRetryable:
    if err := retryPublisher.Publish(ctx, msg, err); err != nil {
        return err
    }
    return commit(ctx, msg)
case ErrorKindNonRetryable:
    if err := dlqPublisher.Publish(ctx, msg, err); err != nil {
        return err
    }
    return commit(ctx, msg)
default:
    return err
}
```

这段代码的核心是：写 retry/DLQ 失败时直接返回错误，不提交原 offset。

---

## 十七、验收清单

- [ ] handler 不直接提交 offset。
- [ ] handler 不直接写 DLQ。
- [ ] 错误分类可以用 `errors.As` 识别。
- [ ] retry/DLQ 消息包含原 topic、partition、offset、payload。
- [ ] retry/DLQ 写失败不会提交原 offset。

---

## 十八、故障用例

至少设计这些测试：

```text
bad-json -> NonRetryable -> DLQ
db-timeout -> Retryable -> retry topic
dlq-publish-failed -> no commit
retry-publish-failed -> no commit
```

这些用例能证明错误分类不是只停留在类型定义。

---

## 十九、最终检查

如果 handler 里出现 `commit`、`publishDLQ` 这类调用，就说明边界可能混乱，应回到 consumer 框架层处理。

---

## 二十、错误分类表

建议在项目文档里维护一张错误分类表：

| 错误类型 | 示例 | 是否重试 | 是否进 DLQ |
| --- | --- | --- | --- |
| 临时错误 | 数据库连接超时 | 是 | 多次失败后进 |
| 业务冲突 | 库存不足 | 否 | 视业务决定 |
| 数据格式错误 | JSON 缺字段 | 否 | 是 |
| 下游限流 | HTTP 429 | 是 | 多次失败后进 |

有了这张表，代码里的 retry 和 DLQ 就不是拍脑袋，而是业务、后端和运维能共同评审的规则。
