# 06. Publish/Subscribe：Fanout Go 实现

## 1. 本节目标

实现发布订阅模式：

```text
fanout_emit -> stage3.logs.fanout
                    -> 临时队列 A -> fanout_receive A
                    -> 临时队列 B -> fanout_receive B
```

特点：

- 生产者发布一条消息。
- 所有正在订阅的消费者都收到一份。
- 每个消费者使用自己的临时队列。

## 2. 创建 fanout 发布者

创建：

```text
cmd/fanout_emit/main.go
```

代码：

```go
package main

import (
	"context"
	"log"
	"os"
	"strings"
	"time"

	amqp "github.com/rabbitmq/amqp091-go"
)

const rabbitURL = "amqp://go_learner:go_learner_pwd@localhost:5672/"
const exchangeName = "stage3.logs.fanout"

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

	err = ch.ExchangeDeclare(exchangeName, "fanout", true, false, false, false, nil)
	if err != nil {
		log.Fatalf("declare exchange: %v", err)
	}

	body := bodyFromArgs()
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	err = ch.PublishWithContext(
		ctx,
		exchangeName,
		"",
		false,
		false,
		amqp.Publishing{
			ContentType: "text/plain",
			Body:        []byte(body),
			Timestamp:   time.Now(),
		},
	)
	if err != nil {
		log.Fatalf("publish log: %v", err)
	}

	log.Printf("sent log: %s", body)
}

func bodyFromArgs() string {
	if len(os.Args) < 2 {
		return "hello fanout"
	}
	return strings.Join(os.Args[1:], " ")
}
```

## 3. 创建 fanout 订阅者

创建：

```text
cmd/fanout_receive/main.go
```

代码：

```go
package main

import (
	"log"

	amqp "github.com/rabbitmq/amqp091-go"
)

const rabbitURL = "amqp://go_learner:go_learner_pwd@localhost:5672/"
const exchangeName = "stage3.logs.fanout"

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

	err = ch.ExchangeDeclare(exchangeName, "fanout", true, false, false, false, nil)
	if err != nil {
		log.Fatalf("declare exchange: %v", err)
	}

	q, err := ch.QueueDeclare(
		"",
		false,
		true,
		true,
		false,
		nil,
	)
	if err != nil {
		log.Fatalf("declare temporary queue: %v", err)
	}

	err = ch.QueueBind(q.Name, "", exchangeName, false, nil)
	if err != nil {
		log.Fatalf("bind queue: %v", err)
	}

	msgs, err := ch.Consume(q.Name, "", true, false, false, false, nil)
	if err != nil {
		log.Fatalf("register consumer: %v", err)
	}

	log.Printf("waiting for fanout logs on queue %s. press Ctrl+C to exit", q.Name)

	forever := make(chan struct{})
	go func() {
		for d := range msgs {
			log.Printf("received log: %s", string(d.Body))
		}
	}()

	<-forever
}
```

## 4. 运行实验

打开两个 PowerShell，分别运行两个订阅者：

```powershell
go run .\cmd\fanout_receive
```

再打开第三个 PowerShell，发布消息：

```powershell
go run .\cmd\fanout_emit "system started"
go run .\cmd\fanout_emit "user registered"
```

预期：

```text
两个 fanout_receive 都能收到同样的消息。
```

## 5. 临时队列解释

这里声明队列时：

```go
ch.QueueDeclare("", false, true, true, false, nil)
```

含义：

```text
name: 空字符串，让 RabbitMQ 自动生成名字
durable: false
autoDelete: true
exclusive: true
```

这个队列适合临时订阅。

消费者退出后，队列会被清理。

## 6. 为什么 fanout 不需要 routing key

发布时：

```go
ch.PublishWithContext(ctx, exchangeName, "", false, false, publishing)
```

routing key 为空。

因为 fanout exchange 的核心规则是：

```text
广播给所有绑定队列。
```

## 7. 适合场景

Fanout 适合：

- 日志广播。
- 临时调试订阅。
- 一个事件所有订阅者都要收到。

如果你需要按事件类型过滤，应该使用 direct 或 topic。

## 8. 本节小结

你已经用 Go 实现 fanout 发布订阅。

下一节实现 direct routing。

