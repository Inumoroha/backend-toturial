# 02. 消费端确认机制：Ack

## 1. Ack 解决什么问题

Ack 是 acknowledgement 的缩写，表示消费者确认消息已经处理完成。

RabbitMQ 需要知道：

```text
消息交给消费者后，消费者到底有没有成功处理？
```

如果成功处理，RabbitMQ 可以删除消息。

如果消费者断开且没有 ack，RabbitMQ 可以重新投递消息。

## 2. Auto Ack 和 Manual Ack

消费消息时，Go 客户端的 `Consume` 方法有一个参数：

```go
autoAck bool
```

### Auto Ack

```go
msgs, err := ch.Consume(queue, consumerTag, true, false, false, false, nil)
```

`autoAck=true` 表示：

```text
RabbitMQ 把消息发给消费者后，就认为消息处理成功。
```

风险：

```text
消费者刚收到消息就崩溃，消息也已经被 RabbitMQ 删除。
```

所以重要业务通常不要使用 auto ack。

### Manual Ack

```go
msgs, err := ch.Consume(queue, consumerTag, false, false, false, false, nil)
```

`autoAck=false` 表示：

```text
消费者必须显式调用 Ack。
```

处理成功后：

```go
if err := d.Ack(false); err != nil {
	log.Printf("ack failed: %v", err)
}
```

这才是重要业务常用方式。

## 3. Ack 的时机

正确顺序：

```text
收到消息
  -> 解析消息
  -> 执行业务逻辑
  -> 业务处理成功
  -> ack
```

不要这样：

```text
收到消息
  -> 先 ack
  -> 再执行业务逻辑
```

否则业务处理失败时，RabbitMQ 已经把消息删除了。

## 4. multiple 参数

Go 中：

```go
d.Ack(false)
```

这里的 `false` 是 `multiple`。

含义：

- `false`：只确认当前这条消息。
- `true`：确认当前 delivery tag 及之前所有未确认消息。

初学和大多数业务处理建议：

```go
d.Ack(false)
```

避免误确认多条消息。

## 5. 手动 Ack 示例

创建：

```text
cmd/ack_worker/main.go
```

代码：

```go
package main

import (
	"log"
	"time"

	amqp "github.com/rabbitmq/amqp091-go"
)

const rabbitURL = "amqp://go_learner:go_learner_pwd@localhost:5672/"
const queueName = "stage4.ack.queue"

func main() {
	conn, err := amqp.Dial(rabbitURL)
	if err != nil {
		log.Fatalf("dial rabbitmq: %v", err)
	}
	defer conn.Close()

	ch, err := conn.Channel()
	if err != nil {
		log.Fatalf("open channel: %v", err)
	}
	defer ch.Close()

	q, err := ch.QueueDeclare(queueName, true, false, false, false, nil)
	if err != nil {
		log.Fatalf("declare queue: %v", err)
	}

	msgs, err := ch.Consume(q.Name, "stage4-ack-worker", false, false, false, false, nil)
	if err != nil {
		log.Fatalf("consume: %v", err)
	}

	log.Println("waiting for messages")

	for d := range msgs {
		log.Printf("received: %s", string(d.Body))

		time.Sleep(2 * time.Second)

		log.Println("business handled successfully")
		if err := d.Ack(false); err != nil {
			log.Printf("ack failed: %v", err)
			continue
		}
		log.Println("acked")
	}
}
```

发布消息可以复用第 3 阶段默认交换机 producer，把队列名改成：

```text
stage4.ack.queue
```

或在 Management UI 中向默认 exchange 发布：

```text
routing key: stage4.ack.queue
payload: hello ack
```

## 6. 观察 Unacked

启动 worker 后发布消息。

在 worker `time.Sleep` 的 2 秒内执行：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers
```

你可能看到：

```text
stage4.ack.queue 0 1 1
```

含义：

- ready 为 0：消息已经投递给消费者。
- unacked 为 1：消费者还没 ack。
- consumers 为 1：有一个消费者。

处理完成后再查：

```text
stage4.ack.queue 0 0 1
```

## 7. 消费者崩溃时会发生什么

如果消费者收到消息但没有 ack，然后进程崩溃：

```text
RabbitMQ 发现连接断开
未 ack 消息重新变为 ready
消息可以被重新投递
```

这就是至少一次投递的基础。

但也意味着：

```text
消费者可能处理了业务，但 ack 前崩溃，消息会被重新投递。
```

所以消费者要做幂等。

## 8. 本节小结

你要记住：

- 重要业务通常使用 manual ack。
- 成功处理后再 ack。
- `Ack(false)` 只确认当前消息。
- 未 ack 消息会显示在 unacked。
- 消费者断开后未 ack 消息会重新投递。
- manual ack 降低丢失风险，但带来重复投递可能。

下一节学习 nack、reject 和 requeue。

