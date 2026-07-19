# 03. Nack、Reject、Requeue：失败处理

## 1. 消费失败时怎么办

消费者处理消息时，不是只有成功。

可能出现：

- JSON 格式错误。
- 数据库超时。
- 外部 API 失败。
- 业务状态不允许。
- 下游服务不可用。

处理失败时，你需要决定：

```text
这条消息要不要重新入队？
要不要丢弃？
要不要进入死信队列？
```

RabbitMQ 提供了：

```go
d.Nack(...)
d.Reject(...)
```

## 2. Ack、Nack、Reject 对比

| 方法 | 含义 | 是否支持 multiple |
| --- | --- | --- |
| `Ack` | 确认成功处理 | 支持 |
| `Nack` | 负确认，表示处理失败 | 支持 |
| `Reject` | 拒绝单条消息 | 不支持 multiple |

常见用法：

```go
d.Ack(false)
d.Nack(false, true)
d.Nack(false, false)
d.Reject(false)
```

## 3. requeue 参数

`Nack` 和 `Reject` 都可以决定是否重新入队。

### 重新入队

```go
d.Nack(false, true)
```

含义：

```text
当前消息处理失败，请 RabbitMQ 重新入队。
```

风险：

```text
如果错误永远无法恢复，消息会反复投递，形成毒消息循环。
```

### 不重新入队

```go
d.Nack(false, false)
```

含义：

```text
当前消息处理失败，不要重新入队。
```

如果队列配置了 dead letter exchange，消息会进入死信交换机。

如果没有 DLX，消息可能被丢弃。

## 4. Reject 和 Nack 的区别

`Reject` 一次只能拒绝一条消息：

```go
d.Reject(false)
```

`Nack` 可以通过 multiple 参数批量拒绝：

```go
d.Nack(true, false)
```

日常业务中，通常使用：

```go
d.Nack(false, false)
```

或：

```go
d.Reject(false)
```

初学建议不要使用 `multiple=true`。

## 5. 错误分类建议

### 格式错误

例如 JSON 解析失败。

通常不应该重试：

```go
d.Nack(false, false)
```

配置了 DLX 时进入死信队列。

### 临时错误

例如数据库连接超时。

可以考虑重试，但不建议简单无限 requeue。

本阶段后面会学习 retry queue：

```text
失败 -> retry queue 延迟 -> 回到业务队列
```

### 业务错误

例如订单状态不合法。

要根据业务判断：

- 可以稍后重试。
- 直接进入死信。
- ack 并记录失败。

## 6. 示例：失败时不 requeue

创建：

```text
cmd/nack_worker/main.go
```

核心代码：

```go
for d := range msgs {
	body := string(d.Body)
	log.Printf("received: %s", body)

	if body == "bad" {
		log.Println("bad message, nack without requeue")
		if err := d.Nack(false, false); err != nil {
			log.Printf("nack failed: %v", err)
		}
		continue
	}

	log.Println("handled successfully")
	if err := d.Ack(false); err != nil {
		log.Printf("ack failed: %v", err)
	}
}
```

如果没有配置 DLX，`bad` 消息会被丢弃。

如果配置了 DLX，`bad` 消息会进入死信队列。

## 7. 示例：失败时 requeue 的风险

核心代码：

```go
if body == "bad" {
	log.Println("bad message, nack with requeue")
	_ = d.Nack(false, true)
	continue
}
```

这会导致：

```text
bad 消息被重新放回队列
再次投递给消费者
再次失败
再次 requeue
```

最终可能让消费者一直处理同一条坏消息。

所以不要把 `Nack(false, true)` 当成默认失败处理。

## 8. 推荐失败处理策略

更健康的策略是：

```text
成功 -> Ack
格式错误 -> Nack(false, false) -> DLQ
临时错误 -> 进入延迟重试队列
超过最大重试次数 -> DLQ
未知错误 -> 记录日志并按策略重试或 DLQ
```

第 9 节会详细实现 retry queue 和 DLQ。

## 9. 本节小结

你要记住：

- `Ack` 表示成功。
- `Nack` / `Reject` 表示失败。
- `requeue=true` 会重新入队，但可能无限循环。
- `requeue=false` 配合 DLX 才能保留失败消息。
- 失败处理要区分错误类型。

下一节学习 prefetch 和消费者限流。

