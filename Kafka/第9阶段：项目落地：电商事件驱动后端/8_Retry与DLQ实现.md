# 8. Retry 与 DLQ 实现

本节目标：为项目实现 retry topic 和 DLQ，确保失败消息不会丢，也不会无限卡住主链路。

---

## 一、Topic

```text
order.created.retry.1m
order.created.retry.5m
order.created.dlq
```

---

## 二、Retry 流程

```text
handler 返回可重试错误
写 retry topic
写成功后提交原 offset
retry consumer 稍后处理
超过次数进入 DLQ
```

---

## 三、DLQ 流程

```text
handler 返回不可重试错误
写 DLQ
写成功后提交原 offset
告警
```

---

## 四、消息结构

Retry 和 DLQ 都要保存：

- original_topic。
- original_partition。
- original_offset。
- original_key。
- error_message。
- payload。
- failed_at。

Retry 还要有：

- attempt。
- next_retry_at。

---

## 五、验收

- 数据库临时错误进入 retry。
- JSON 错误进入 DLQ。
- 写 retry 失败不提交 offset。
- 写 DLQ 失败不提交 offset。

---

## 六、错误类型

`internal/kafka/error.go`：

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

## 七、Consumer 错误处理

```go
err := handler.Handle(ctx, msg)
if err == nil {
    commit()
    return
}

switch Classify(err) {
case ErrorKindRetryable:
    if publishRetry(ctx, msg, err) == nil {
        commit()
    }
case ErrorKindNonRetryable:
    if publishDLQ(ctx, msg, err) == nil {
        commit()
    }
}
```

---

## 八、Retry Struct

```go
type RetryMessage struct {
    EventID           string          `json:"event_id"`
    OriginalTopic     string          `json:"original_topic"`
    OriginalPartition int32           `json:"original_partition"`
    OriginalOffset    int64           `json:"original_offset"`
    OriginalKey       string          `json:"original_key"`
    Attempt           int             `json:"attempt"`
    LastError         string          `json:"last_error"`
    NextRetryAt       time.Time       `json:"next_retry_at"`
    Payload           json.RawMessage `json:"payload"`
}
```

---

## 九、DLQ Struct

```go
type DLQMessage struct {
    OriginalTopic     string          `json:"original_topic"`
    OriginalPartition int32           `json:"original_partition"`
    OriginalOffset    int64           `json:"original_offset"`
    OriginalKey       string          `json:"original_key"`
    ErrorType         string          `json:"error_type"`
    ErrorMessage      string          `json:"error_message"`
    FailedAt          time.Time       `json:"failed_at"`
    Payload           json.RawMessage `json:"payload"`
}
```

---

## 十、验证场景

### JSON 错误

发送：

```text
bad-json
```

预期：

```text
进入 order.created.dlq
原 offset 被提交
```

### 数据库错误

临时停止数据库或让 repository 返回错误。

预期：

```text
进入 order.created.retry.1m
原 offset 被提交
```

---

## 十一、本节练习

1. 实现错误分类。
2. 实现 retry message 构造。
3. 实现 DLQ message 构造。
4. 验证 JSON 错误进入 DLQ。
5. 验证数据库错误进入 retry。

---

## 十二、错误分类函数

前面定义了 `HandlerError`，还需要一个统一分类函数：

```go
func Classify(err error) ErrorKind {
    if err == nil {
        return ""
    }

    var handlerErr HandlerError
    if errors.As(err, &handlerErr) {
        return handlerErr.Kind
    }

    return ErrorKindRetryable
}
```

默认按可重试处理，是因为数据库超时、网络抖动、下游短暂不可用都属于暂时性问题。只有明确知道消息本身坏了，才标记为不可重试。

典型不可重试错误：

```text
JSON 无法解析
必填字段缺失
event_type 不支持
version 不兼容
```

典型可重试错误：

```text
数据库连接失败
事务提交超时
Kafka 暂时不可用
外部服务 503
```

---

## 十三、Retry 次数设计

不要只设计一个 retry topic。推荐第一版使用两级：

```text
order.created.retry.1m
order.created.retry.5m
```

处理规则：

| attempt | 进入 topic | 说明 |
| --- | --- | --- |
| 1 | `order.created.retry.1m` | 第一次失败，稍后重试 |
| 2 | `order.created.retry.5m` | 第二次失败，拉开间隔 |
| 3+ | `order.created.dlq` | 多次失败，交给人工排查 |

学习项目里可以不用真的等待 1 分钟和 5 分钟，可以先用 `10s`、`30s` 验证流程。

---

## 十四、构造 Retry 消息

```go
func BuildRetryMessage(msg kafka.ConsumedMessage, err error, attempt int, delay time.Duration) RetryMessage {
    return RetryMessage{
        EventID:           extractEventID(msg.Value),
        OriginalTopic:     msg.Topic,
        OriginalPartition: msg.Partition,
        OriginalOffset:    msg.Offset,
        OriginalKey:       string(msg.Key),
        Attempt:           attempt,
        LastError:         err.Error(),
        NextRetryAt:       time.Now().Add(delay),
        Payload:           append([]byte(nil), msg.Value...),
    }
}
```

注意 `Payload` 要保存原始消息。不要只保存错误信息，否则后面无法重新处理。

---

## 十五、构造 DLQ 消息

```go
func BuildDLQMessage(msg kafka.ConsumedMessage, err error, errorType string) DLQMessage {
    return DLQMessage{
        OriginalTopic:     msg.Topic,
        OriginalPartition: msg.Partition,
        OriginalOffset:    msg.Offset,
        OriginalKey:       string(msg.Key),
        ErrorType:         errorType,
        ErrorMessage:      err.Error(),
        FailedAt:          time.Now(),
        Payload:           append([]byte(nil), msg.Value...),
    }
}
```

DLQ 消息要能回答四个问题：

```text
原消息从哪里来？
原消息内容是什么？
为什么失败？
失败发生在什么时候？
```

只有这样，后续排查才不会变成猜谜。

---

## 十六、Consumer 处理模板

主 topic consumer 可以采用下面的结构：

```go
func (c *Consumer) handleMessage(ctx context.Context, msg kafka.ConsumedMessage) {
    err := c.handler.Handle(ctx, msg)
    if err == nil {
        c.commit(ctx, msg)
        return
    }

    switch kafka.Classify(err) {
    case kafka.ErrorKindNonRetryable:
        if pubErr := c.publisher.PublishDLQ(ctx, msg, err); pubErr != nil {
            c.logger.Error("publish dlq failed", "error", pubErr)
            return
        }
        c.commit(ctx, msg)

    case kafka.ErrorKindRetryable:
        if pubErr := c.publisher.PublishRetry(ctx, msg, err); pubErr != nil {
            c.logger.Error("publish retry failed", "error", pubErr)
            return
        }
        c.commit(ctx, msg)
    }
}
```

关键点：

```text
写 retry/DLQ 成功以后，才能提交原消息 offset。
写 retry/DLQ 失败时，不提交 offset。
```

这样可以保证失败消息至少还留在原 topic，后续仍有机会处理。

---

## 十七、Retry Consumer 怎么等到 next_retry_at

学习项目里可以用简单判断：

```go
if time.Now().Before(retryMsg.NextRetryAt) {
    time.Sleep(time.Until(retryMsg.NextRetryAt))
}
```

这能帮助理解流程，但生产里要谨慎。因为 consumer 睡眠期间会占住分区，可能影响 rebalance。

更稳的方向是：

```text
短延迟：retry topic + consumer 控制。
长延迟：数据库延迟表或专门的延迟队列。
```

当前教程先实现最小可用版本即可。

---

## 十八、验证命令

发送一条坏 JSON：

```bash
printf 'bad-json\n' | kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic order.created
```

查看 DLQ：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic order.created.dlq \
  --from-beginning \
  --max-messages 1
```

模拟数据库错误时，查看 retry topic：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic order.created.retry.1m \
  --from-beginning \
  --max-messages 1
```

再看 consumer group offset：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group inventory-service
```

预期：

```text
原 topic 的 offset 已提交。
retry 或 DLQ topic 中能看到失败消息。
日志里能看到 original_topic、partition、offset。
```

---

## 十九、常见错误

### 1. 写 DLQ 前就提交 offset

如果 DLQ 写入失败，原消息 offset 又已经提交，这条消息就消失了。

### 2. DLQ 只保存错误，不保存 payload

没有 payload 就无法重放，也无法判断是消息格式问题还是代码问题。

### 3. 所有错误都 retry

坏 JSON 重试多少次都没用，会浪费资源并污染监控。

### 4. retry 没有最大次数

没有最大次数会让问题消息在系统里永久循环。DLQ 的意义就是把“自动处理不了”的消息显式暴露出来。

---

## 二十、面试表达

可以这样描述你的失败处理方案：

```text
consumer handler 返回错误后，我会先做错误分类。格式错误、字段缺失这类不可重试错误进入 DLQ；数据库超时这类暂时性错误进入 retry topic。
只有当 retry 或 DLQ 消息发布成功后，才提交原消息 offset。
这样主 topic 不会被坏消息长期卡住，同时失败消息也不会静默丢失。
```
