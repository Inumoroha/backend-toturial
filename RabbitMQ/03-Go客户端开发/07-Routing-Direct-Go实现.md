# 07. Routing：Direct Go 实现

## 1. 本节目标

实现 direct routing：

```text
direct_emit -> stage3.notify.direct
                    -- email.send --> email receiver
                    -- sms.send   --> sms receiver
```

特点：

- 生产者发布时指定 routing key。
- 消费者按 binding key 订阅。
- direct exchange 精确匹配。

## 2. 创建 direct 发布者

创建：

```text
cmd/direct_emit/main.go
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
const exchangeName = "stage3.notify.direct"

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

	err = ch.ExchangeDeclare(exchangeName, "direct", true, false, false, false, nil)
	if err != nil {
		log.Fatalf("declare exchange: %v", err)
	}

	key := routingKeyFromArgs()
	body := bodyFromArgs()

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	err = ch.PublishWithContext(
		ctx,
		exchangeName,
		key,
		false,
		false,
		amqp.Publishing{
			ContentType: "text/plain",
			Body:        []byte(body),
			Timestamp:   time.Now(),
		},
	)
	if err != nil {
		log.Fatalf("publish message: %v", err)
	}

	log.Printf("sent %q with routing key %q", body, key)
}

func routingKeyFromArgs() string {
	if len(os.Args) < 2 {
		return "email.send"
	}
	return os.Args[1]
}

func bodyFromArgs() string {
	if len(os.Args) < 3 {
		return "hello direct"
	}
	return strings.Join(os.Args[2:], " ")
}
```

运行示例：

```powershell
go run .\cmd\direct_emit email.send "send email to user"
go run .\cmd\direct_emit sms.send "send sms to user"
```

## 3. 创建 direct 接收者

创建：

```text
cmd/direct_receive/main.go
```

代码：

```go
package main

import (
	"log"
	"os"

	amqp "github.com/rabbitmq/amqp091-go"
)

const rabbitURL = "amqp://go_learner:go_learner_pwd@localhost:5672/"
const exchangeName = "stage3.notify.direct"

func main() {
	if len(os.Args) < 2 {
		log.Fatalf("usage: go run .\\cmd\\direct_receive [binding_key...]")
	}

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

	err = ch.ExchangeDeclare(exchangeName, "direct", true, false, false, false, nil)
	if err != nil {
		log.Fatalf("declare exchange: %v", err)
	}

	q, err := ch.QueueDeclare("", false, true, true, false, nil)
	if err != nil {
		log.Fatalf("declare temporary queue: %v", err)
	}

	for _, key := range os.Args[1:] {
		err = ch.QueueBind(q.Name, key, exchangeName, false, nil)
		if err != nil {
			log.Fatalf("bind queue with key %q: %v", key, err)
		}
		log.Printf("bound queue %s with key %s", q.Name, key)
	}

	msgs, err := ch.Consume(q.Name, "", true, false, false, false, nil)
	if err != nil {
		log.Fatalf("register consumer: %v", err)
	}

	log.Println("waiting for direct messages. press Ctrl+C to exit")

	forever := make(chan struct{})
	go func() {
		for d := range msgs {
			log.Printf("routing_key=%s body=%s", d.RoutingKey, string(d.Body))
		}
	}()

	<-forever
}
```

## 4. 运行实验

打开第一个接收者，只订阅邮件：

```powershell
go run .\cmd\direct_receive email.send
```

打开第二个接收者，只订阅短信：

```powershell
go run .\cmd\direct_receive sms.send
```

打开第三个 PowerShell 发布：

```powershell
go run .\cmd\direct_emit email.send "welcome email"
go run .\cmd\direct_emit sms.send "login sms"
go run .\cmd\direct_emit push.send "push notification"
```

预期：

- `email.send` 只被邮件接收者收到。
- `sms.send` 只被短信接收者收到。
- `push.send` 没有匹配绑定，接收者收不到。

## 5. 一个消费者绑定多个 key

运行：

```powershell
go run .\cmd\direct_receive email.send sms.send
```

它会同时收到：

```text
email.send
sms.send
```

因为同一个 queue 可以绑定多个 binding key。

## 6. Direct 的核心规则

Direct exchange 的规则是：

```text
routing key 精确匹配 binding key
```

它不是只能一对一。

如果多个队列都绑定 `email.send`，它们都会收到消息。

## 7. 本节小结

你已经用 Go 实现 direct routing。

下一节实现 topic routing。

