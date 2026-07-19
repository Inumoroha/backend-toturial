# 07. Mandatory Publish 与不可路由消息

## 1. 什么是不可路由消息

消息发布到 exchange 后，如果没有匹配到任何 queue，就叫不可路由。

例如：

```text
Exchange: stage4.return.direct
Binding: email.send -> stage4.email.queue
Publish routing key: sms.send
```

`sms.send` 没有匹配 binding，消息无法进入队列。

## 2. 默认情况下会怎样

默认情况下，如果消息不可路由，RabbitMQ 可能直接丢弃它。

这很危险：

```text
生产者 Publish 返回 nil
但业务队列没有收到消息
```

所以重要业务需要处理不可路由消息。

## 3. mandatory 参数

Go 发布方法中有参数：

```go
PublishWithContext(ctx, exchange, key, mandatory, immediate, msg)
```

把 `mandatory` 设置为 `true`：

```go
err := ch.PublishWithContext(ctx, exchangeName, routingKey, true, false, msg)
```

如果消息不可路由，RabbitMQ 会把消息 return 给生产者。

生产者需要监听：

```go
ch.NotifyReturn(...)
```

## 4. Return 示例

创建：

```text
cmd/mandatory_publisher/main.go
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
const exchangeName = "stage4.return.direct"
const queueName = "stage4.return.email.queue"

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

	returns := ch.NotifyReturn(make(chan amqp.Return, 1))

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	err = ch.PublishWithContext(
		ctx,
		exchangeName,
		"sms.send",
		true,
		false,
		amqp.Publishing{
			ContentType: "text/plain",
			MessageId:   "return-msg-001",
			Body:        []byte("this message is unroutable"),
		},
	)
	if err != nil {
		log.Fatalf("publish failed: %v", err)
	}

	select {
	case ret := <-returns:
		log.Printf("message returned: replyCode=%d replyText=%s exchange=%s routingKey=%s body=%s",
			ret.ReplyCode,
			ret.ReplyText,
			ret.Exchange,
			ret.RoutingKey,
			string(ret.Body),
		)
	case <-time.After(2 * time.Second):
		log.Println("no return received")
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
	return ch.QueueBind(q.Name, "email.send", exchangeName, false, nil)
}
```

运行：

```powershell
go run .\cmd\mandatory_publisher
```

预期：

```text
message returned: replyCode=312 replyText=NO_ROUTE ...
```

因为发布的是：

```text
routing key: sms.send
```

但只绑定了：

```text
email.send
```

## 5. mandatory 和 publisher confirm 的关系

它们解决的问题不同：

| 机制 | 解决什么 |
| --- | --- |
| publisher confirm | broker 是否确认接收发布 |
| mandatory return | 消息是否无法路由到队列 |

重要业务通常两个都要考虑：

```text
Publish mandatory=true
开启 publisher confirm
监听 NotifyReturn
等待 NotifyPublish
```

## 6. 不可路由消息如何处理

常见策略：

- 记录错误日志。
- 告警。
- 保存到数据库 outbox，等待人工或自动修复。
- 修正 binding 后重新发布。
- 阻止上线错误拓扑。

不要悄悄忽略 return。

## 7. immediate 参数

Go API 中还有 `immediate` 参数。

RabbitMQ 不支持 immediate 语义，通常应设置为：

```go
false
```

学习和业务代码都不要设置为 true。

## 8. 本节小结

你要记住：

- 发布成功不代表消息进入了队列。
- routing key 不匹配会导致消息不可路由。
- `mandatory=true` 可以让不可路由消息 return 给生产者。
- 使用 `NotifyReturn` 监听返回消息。
- mandatory 和 publisher confirm 要配合理解。

下一节学习消费者幂等。

