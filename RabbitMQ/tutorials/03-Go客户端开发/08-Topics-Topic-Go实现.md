# 08. Topics：Topic Go 实现

## 1. 本节目标

实现 topic routing：

```text
topic_emit -> stage3.business.topic
                  -- order.#     --> order receiver
                  -- *.created   --> created receiver
                  -- payment.*   --> payment receiver
```

特点：

- routing key 使用点分隔。
- binding key 支持 `*` 和 `#`。
- 适合业务事件总线。

## 2. Topic 规则回顾

| 通配符 | 含义 |
| --- | --- |
| `*` | 匹配一个单词 |
| `#` | 匹配零个或多个单词 |

示例：

```text
order.*       -> order.created、order.paid
order.#       -> order.created、order.payment.succeeded
*.created     -> order.created、user.created
payment.*     -> payment.succeeded、payment.failed
```

## 3. 创建 topic 发布者

创建：

```text
cmd/topic_emit/main.go
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
const exchangeName = "stage3.business.topic"

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

	err = ch.ExchangeDeclare(exchangeName, "topic", true, false, false, false, nil)
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
			Type:        key,
			Body:        []byte(body),
			Timestamp:   time.Now(),
		},
	)
	if err != nil {
		log.Fatalf("publish topic message: %v", err)
	}

	log.Printf("sent routing_key=%s body=%s", key, body)
}

func routingKeyFromArgs() string {
	if len(os.Args) < 2 {
		return "order.created"
	}
	return os.Args[1]
}

func bodyFromArgs() string {
	if len(os.Args) < 3 {
		return "hello topic"
	}
	return strings.Join(os.Args[2:], " ")
}
```

## 4. 创建 topic 接收者

创建：

```text
cmd/topic_receive/main.go
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
const exchangeName = "stage3.business.topic"

func main() {
	if len(os.Args) < 2 {
		log.Fatalf("usage: go run .\\cmd\\topic_receive [binding_key...]")
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

	err = ch.ExchangeDeclare(exchangeName, "topic", true, false, false, false, nil)
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
		log.Printf("bound queue %s with pattern %s", q.Name, key)
	}

	msgs, err := ch.Consume(q.Name, "", true, false, false, false, nil)
	if err != nil {
		log.Fatalf("register consumer: %v", err)
	}

	log.Println("waiting for topic messages. press Ctrl+C to exit")

	forever := make(chan struct{})
	go func() {
		for d := range msgs {
			log.Printf("routing_key=%s body=%s", d.RoutingKey, string(d.Body))
		}
	}()

	<-forever
}
```

## 5. 运行实验

打开第一个接收者，订阅所有订单事件：

```powershell
go run .\cmd\topic_receive "order.#"
```

打开第二个接收者，订阅所有 created 事件：

```powershell
go run .\cmd\topic_receive "*.created"
```

打开第三个接收者，订阅 payment 一级事件：

```powershell
go run .\cmd\topic_receive "payment.*"
```

发布消息：

```powershell
go run .\cmd\topic_emit order.created "order 1001 created"
go run .\cmd\topic_emit order.payment.succeeded "order 1001 payment succeeded"
go run .\cmd\topic_emit user.created "user 2001 created"
go run .\cmd\topic_emit payment.failed "payment failed"
```

预期：

- `order.created` 被 `order.#` 和 `*.created` 收到。
- `order.payment.succeeded` 被 `order.#` 收到。
- `user.created` 被 `*.created` 收到。
- `payment.failed` 被 `payment.*` 收到。

## 6. Topic 适合业务事件

推荐 routing key：

```text
order.created
order.paid
order.cancelled
payment.succeeded
payment.failed
user.registered
```

不推荐：

```text
created
paid
test
abc
```

好的 routing key 应该表达业务事实。

## 7. 本节小结

你已经用 Go 实现 topic routing。

下一节学习消息结构、JSON、properties 和 headers。

