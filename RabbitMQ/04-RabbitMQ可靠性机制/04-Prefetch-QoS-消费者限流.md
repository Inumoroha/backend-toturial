# 04. Prefetch / QoS：消费者限流

## 1. Prefetch 解决什么问题

如果消费者开启 manual ack，RabbitMQ 可以一次性投递多条消息给消费者。

如果没有限制，可能出现：

```text
消费者一次拿到大量消息
消息都处于 unacked
其他消费者拿不到任务
单个消费者内存压力变大
处理慢的消费者拖住很多消息
```

Prefetch 用来限制：

```text
一个消费者最多同时持有多少条未 ack 消息。
```

## 2. Go 中设置 QoS

使用：

```go
err := ch.Qos(
	1,
	0,
	false,
)
```

参数含义：

```text
prefetchCount: 1
prefetchSize: 0
global: false
```

常用写法：

```go
if err := ch.Qos(1, 0, false); err != nil {
	log.Fatalf("set qos: %v", err)
}
```

## 3. prefetchCount

`prefetchCount=1` 表示：

```text
这个消费者同一时间最多拿 1 条未 ack 消息。
```

处理完并 ack 后，RabbitMQ 再投递下一条。

如果设置为 10：

```text
这个消费者最多同时持有 10 条未 ack 消息。
```

## 4. RabbitMQ 中 global=false 的含义

在 RabbitMQ 中，常见使用是：

```go
ch.Qos(prefetchCount, 0, false)
```

`global=false` 时，prefetch 通常按消费者应用。

也就是同一个 channel 上每个 consumer 有自己的 prefetch 限制。

学习阶段建议一直使用：

```go
ch.Qos(1, 0, false)
```

或：

```go
ch.Qos(5, 0, false)
```

## 5. 为什么 Work Queue 常用 prefetch=1

假设有两个 worker：

```text
worker-1 处理很慢
worker-2 处理很快
```

如果 RabbitMQ 一次给 worker-1 很多消息，这些消息会一直 unacked，worker-2 可能空闲。

设置：

```go
ch.Qos(1, 0, false)
```

可以让 RabbitMQ 更公平地分发任务：

```text
worker 处理完一条，再拿下一条。
```

## 6. prefetch 不是越小越好

`prefetch=1` 公平，但吞吐可能不高。

如果每条消息处理很快，网络往返成本会变明显。

可以尝试：

```text
prefetch=5
prefetch=10
prefetch=50
```

选择要看：

- 单条消息处理耗时。
- 消费者并发能力。
- 消费者内存。
- 下游数据库承载能力。
- 业务是否要求公平。

## 7. 示例：设置 prefetch

消费者代码关键部分：

```go
if err := ch.Qos(3, 0, false); err != nil {
	log.Fatalf("set qos: %v", err)
}

msgs, err := ch.Consume(
	queueName,
	"stage4-prefetch-worker",
	false,
	false,
	false,
	false,
	nil,
)
if err != nil {
	log.Fatalf("consume: %v", err)
}
```

然后每条消息处理 5 秒：

```go
for d := range msgs {
	log.Printf("received: %s", string(d.Body))
	time.Sleep(5 * time.Second)
	_ = d.Ack(false)
}
```

观察：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers
```

你会看到 unacked 最多接近 prefetch 数量。

## 8. prefetch 和 goroutine 并发

如果消费者内部开多个 goroutine 并发处理消息，prefetch 要和 worker 数匹配。

例如：

```text
consumer 内部 5 个 worker goroutine
```

可以考虑：

```go
ch.Qos(5, 0, false)
```

但要小心：

- 每个消息处理完成后再 ack。
- 不要并发使用同一个 delivery 做重复 ack。
- 出错时要清晰处理 nack/retry。

初学阶段建议：

```text
一个 consumer 顺序处理，先理解 prefetch。
```

## 9. 常见设置建议

| 场景 | 建议起点 |
| --- | --- |
| 任务耗时较长 | `prefetch=1` |
| 任务耗时较短 | `prefetch=10` |
| 消费者内部并发 5 个 worker | `prefetch=5` 或略高 |
| 下游数据库压力大 | 较小 prefetch |
| 追求高吞吐 | 压测后逐步调大 |

## 10. 本节小结

你要记住：

- prefetch 限制未 ack 消息数量。
- manual ack 场景才特别有意义。
- Work Queue 常从 `prefetch=1` 开始。
- prefetch 太大可能导致 unacked 堆积。
- prefetch 太小可能限制吞吐。
- 需要结合业务耗时和下游能力调优。

下一节学习持久化链路。

