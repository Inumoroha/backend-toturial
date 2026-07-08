# 06. Publisher Confirm：生产端确认

## 1. Publisher Confirm 解决什么问题

生产者调用：

```go
ch.PublishWithContext(...)
```

返回 `nil` 只能说明：

```text
客户端把发布命令写出去了。
```

它不等于：

```text
RabbitMQ 已经可靠接收并处理了这条消息。
```

Publisher confirm 用来让 RabbitMQ 告诉生产者：

```text
这条发布的消息，我确认了。
```

这是生产端可靠性的核心机制。

## 2. Confirm 模式基本流程

生产者流程：

```text
打开 channel
启用 confirm mode
发布消息
等待 RabbitMQ confirm
收到 ack -> 发布成功
收到 nack 或超时 -> 发布失败或未知，按策略重试
```

Go 中启用：

```go
if err := ch.Confirm(false); err != nil {
	log.Fatalf("enable confirm mode: %v", err)
}
```

## 3. 简单同步 confirm 示例

创建：

```text
cmd/confirm_publisher/main.go
```

代码：

```go
package main

import (
	"context"
	"log"
	"time"

	amqp "github.com/rabbitmq/amqp091-go"
)

const rabbitURL = "amqp://go_learner:go_learner_pwd@localhost:5672/"
const exchangeName = "stage4.confirm.direct"
const queueName = "stage4.confirm.queue"
const routingKey = "confirm.task"

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

	if err := declareTopology(ch); err != nil {
		log.Fatalf("declare topology: %v", err)
	}

	if err := ch.Confirm(false); err != nil {
		log.Fatalf("enable confirm mode: %v", err)
	}

	confirms := ch.NotifyPublish(make(chan amqp.Confirmation, 1))

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	err = ch.PublishWithContext(
		ctx,
		exchangeName,
		routingKey,
		false,
		false,
		amqp.Publishing{
			ContentType:  "text/plain",
			DeliveryMode: amqp.Persistent,
			MessageId:    "confirm-msg-001",
			Timestamp:    time.Now(),
			Body:         []byte("hello publisher confirm"),
		},
	)
	if err != nil {
		log.Fatalf("publish failed: %v", err)
	}

	select {
	case confirm := <-confirms:
		if confirm.Ack {
			log.Printf("message confirmed by broker, deliveryTag=%d", confirm.DeliveryTag)
			return
		}
		log.Fatalf("message nacked by broker, deliveryTag=%d", confirm.DeliveryTag)
	case <-time.After(5 * time.Second):
		log.Fatalf("confirm timeout")
	}
}

func declareTopology(ch *amqp.Channel) error {
	if err := ch.ExchangeDeclare(exchangeName, "direct", true, false, false, false, nil); err != nil {
		return err
	}
	q, err := ch.QueueDeclare(queueName, true, false, false, false, nil)
	if err != nil {
		return err
	}
	return ch.QueueBind(q.Name, routingKey, exchangeName, false, nil)
}
```

运行：

```powershell
go run .\cmd\confirm_publisher
```

预期：

```text
message confirmed by broker
```

## 4. Confirm ack 表示什么

Confirm ack 表示 RabbitMQ 已经确认这条消息。

对于持久化消息和 durable queue，确认通常意味着 RabbitMQ 已经把消息处理到足以承担它承诺的可靠性级别。

但你仍要理解：

```text
Confirm ack 不是消费者处理成功。
```

它只覆盖生产者到 RabbitMQ 这一段。

消费者处理成功要靠：

```text
manual ack
```

## 5. Confirm 超时怎么办

如果等待 confirm 超时，消息处于未知状态：

```text
可能 broker 收到了，但 confirm 响应丢了。
可能 broker 没收到。
```

生产者通常会重试。

这意味着可能产生重复消息。

所以消费者必须幂等。

## 6. 批量 confirm 思路

简单同步 confirm 每发一条等一次，容易慢。

更高吞吐可以：

- 批量发布 N 条。
- 等待 N 个 confirm。
- 维护 delivery tag 和 message id 的映射。
- 失败或超时后重试未确认消息。

本阶段先掌握单条同步 confirm。

生产级 publisher 后续可以封装异步 confirm。

## 7. 常见错误

### 忘记启用 confirm mode

必须先调用：

```go
ch.Confirm(false)
```

再使用：

```go
ch.NotifyPublish(...)
```

### confirm channel 缓冲太小

高吞吐时 confirm channel 缓冲太小可能影响处理。

学习阶段单条发布用：

```go
make(chan amqp.Confirmation, 1)
```

批量发布要根据并发量调整。

### 把 confirm 当成消费成功

Confirm 只说明 broker 确认了消息，不说明消费者处理了消息。

## 8. 本节小结

你要记住：

- Publisher confirm 是生产端可靠性的关键。
- `PublishWithContext` 成功不等于 broker confirm。
- confirm ack 不等于消费者处理成功。
- confirm 超时后消息状态未知，重试可能导致重复。
- 消费者幂等仍然必须做。

下一节学习 mandatory publish 和不可路由消息。

