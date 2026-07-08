# 03 封装 Go Consumer Group

本节目标：设计一个支持手动提交、错误分类、优雅退出的 Go consumer group。

## Consumer 封装目标

一个生产可用的 consumer wrapper 应该支持：

- 指定 group id。
- 订阅一个或多个 topic。
- 手动提交 offset。
- handler 插拔。
- 错误分类。
- retry topic 和 DLQ。
- context 取消。
- graceful shutdown。
- 指标采集。

## Handler 接口

推荐把业务处理抽象成接口：

```go
type Message struct {
    Topic     string
    Partition int32
    Offset    int64
    Key       []byte
    Value     []byte
    Headers   map[string]string
}

type Handler interface {
    Handle(ctx context.Context, msg Message) error
}
```

consumer wrapper 负责 Kafka 细节，handler 只关心业务。

## 消费流程

```text
poll message
decode metadata
start timer
handler.Handle(ctx, msg)
if success:
    commit offset
if retryable error:
    send retry topic
    commit offset after retry message sent
if non-retryable error:
    send DLQ
    commit offset after DLQ message sent
record metrics
```

## 错误分类接口

```go
type RetryableError interface {
    error
    Retryable() bool
}
```

更简单的方式是定义函数：

```go
func IsRetryable(err error) bool {
    return errors.Is(err, ErrDatabaseTemporary) ||
        errors.Is(err, ErrRemoteTimeout)
}
```

## 提交 offset 的边界

必须反复强调：

- handler 成功后提交。
- retry 消息写入成功后提交。
- DLQ 消息写入成功后提交。
- 无法确认消息去向时不提交。

## 优雅退出流程

```text
收到 SIGTERM
取消 root context
停止 poll 新消息
等待当前 handler 完成或超时
提交已成功处理的 offset
关闭 consumer
关闭 producer
退出
```

建议为退出设置最大等待时间，例如 30 秒。

## 并发处理建议

初学阶段先做顺序消费。

如果必须并发：

- 不要破坏同一 partition 内的提交顺序。
- 每个 partition 维护独立 worker。
- offset 只能提交连续成功处理的最大位置。
- 不要让 goroutine 无限增长。

## 本节练习

1. 设计 `Handler` 接口。
2. 写一个 `OrderCreatedHandler` 的伪代码。
3. 写出 consumer 收到 SIGTERM 后的退出步骤。
4. 思考：为什么并发处理会让 offset 提交变复杂？

