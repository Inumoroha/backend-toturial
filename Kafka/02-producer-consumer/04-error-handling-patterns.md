# 04 消费错误处理模式

本节目标：建立生产可用的错误处理思路。Kafka consumer 的质量，不取决于正常消息能不能处理，而取决于异常消息怎么处理。

## 错误处理的核心原则

不要让一条坏消息无限卡住整个 partition。

但是也不能遇到错误就提交 offset，否则可能丢失可恢复的业务。

正确做法是先分类：

- 可重试错误：稍后再处理。
- 不可重试错误：进入死信并提交原消息。
- 需要人工介入：进入死信并报警。

## 本地重试

适合非常短暂的错误。

例如数据库连接池偶发抖动，可以在 consumer 内重试 2 到 3 次。

不适合：

- 下游长时间不可用。
- 单条消息业务状态不满足。
- JSON 或 schema 错误。

本地重试时间不能太长，否则会阻塞 partition。

## Retry Topic

retry topic 用来延迟或分层重试。

示例：

```text
order.created
order.created.retry.1m
order.created.retry.5m
order.created.retry.30m
order.created.dlq
```

流程：

1. 主 consumer 处理失败。
2. 判断是可重试错误。
3. 写入 retry topic，并带上 attempt。
4. 提交原消息 offset。
5. retry consumer 稍后再处理。
6. 超过最大次数进入 DLQ。

## Dead Letter Topic

dead letter topic 保存无法继续处理的消息。

DLQ 消息应包含：

- 原 topic。
- 原 partition。
- 原 offset。
- 原 key。
- 原 value。
- 错误原因。
- 失败时间。
- attempt 次数。
- handler 名称。

这样后续才能人工排查或修复后重放。

## 什么时候提交 offset

| 场景 | 是否提交原消息 offset |
| --- | --- |
| 业务处理成功 | 提交 |
| 写入 retry topic 成功 | 提交 |
| 写入 DLQ 成功 | 提交 |
| 临时错误但还没转移消息 | 不提交 |
| 进程即将退出且消息未处理完 | 不提交 |

关键点：如果你决定不再立即处理原消息，必须先把它可靠转移到 retry 或 DLQ，再提交 offset。

## Go 后端处理伪代码

```go
msg := consumer.Poll(ctx)

err := handle(msg)
if err == nil {
    consumer.Commit(msg)
    return
}

if isRetryable(err) {
    retryErr := retryProducer.Send(ctx, buildRetryMessage(msg, err))
    if retryErr == nil {
        consumer.Commit(msg)
    }
    return
}

dlqErr := dlqProducer.Send(ctx, buildDLQMessage(msg, err))
if dlqErr == nil {
    consumer.Commit(msg)
}
```

这段伪代码强调的是顺序：先处理或转移成功，再提交 offset。

## 本节练习

1. 为 `order.created` 设计 3 个 retry topic。
2. 写出 DLQ 消息应该包含的字段。
3. 思考：如果写 DLQ 失败，能不能提交原消息 offset？
4. 思考：如果业务处理成功但提交 offset 失败，会发生什么？

