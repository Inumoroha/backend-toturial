# 04. 声明 Exchange、Queue、Binding

## 1. 本节目标

上一节使用默认 exchange。

本节改为显式声明：

```text
Exchange: stage3.events.topic
Queue: stage3.order.created.queue
Binding: order.created
```

消息流：

```text
producer -> stage3.events.topic -- order.created --> stage3.order.created.queue -> consumer
```

## 2. 为什么要在代码里声明拓扑

RabbitMQ 资源可以在 UI 中手动创建，也可以由应用启动时声明。

学习阶段推荐在代码里声明：

- 代码更可重复。
- demo 不依赖手工点击。
- 更容易理解 API 和模型关系。

生产环境是否由应用自动声明，要看团队规范。有些团队会使用 Terraform、Helm、脚本或运维平台提前创建。

## 3. 创建声明程序

创建：

```text
cmd/declare_topology/main.go
```

代码：

```go
package main

import (
	"log"

	amqp "github.com/rabbitmq/amqp091-go"
)

const rabbitURL = "amqp://go_learner:go_learner_pwd@localhost:5672/"

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

	const exchangeName = "stage3.events.topic"
	const queueName = "stage3.order.created.queue"
	const routingKey = "order.created"

	err = ch.ExchangeDeclare(
		exchangeName,
		"topic",
		true,
		false,
		false,
		false,
		nil,
	)
	if err != nil {
		log.Fatalf("declare exchange: %v", err)
	}

	q, err := ch.QueueDeclare(
		queueName,
		true,
		false,
		false,
		false,
		nil,
	)
	if err != nil {
		log.Fatalf("declare queue: %v", err)
	}

	err = ch.QueueBind(
		q.Name,
		routingKey,
		exchangeName,
		false,
		nil,
	)
	if err != nil {
		log.Fatalf("bind queue: %v", err)
	}

	log.Println("topology declared")
}
```

运行：

```powershell
go run .\cmd\declare_topology
```

## 4. 用命令检查

查看 exchange：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_exchanges name type durable auto_delete
```

查看 queue：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_queues name durable messages_ready consumers
```

查看 binding：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_bindings
```

你应该能看到：

```text
stage3.events.topic
stage3.order.created.queue
order.created
```

## 5. 发布一条 order.created 消息

创建：

```text
cmd/event_producer/main.go
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

	const exchangeName = "stage3.events.topic"
	const routingKey = "order.created"

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	body := `{"message_id":"order-msg-001","event_type":"order.created","order_id":1001}`
	err = ch.PublishWithContext(
		ctx,
		exchangeName,
		routingKey,
		false,
		false,
		amqp.Publishing{
			ContentType:  "application/json",
			DeliveryMode: amqp.Persistent,
			Type:         "order.created",
			MessageId:    "order-msg-001",
			Timestamp:    time.Now(),
			Body:         []byte(body),
		},
	)
	if err != nil {
		log.Fatalf("publish event: %v", err)
	}

	log.Println("published order.created")
}
```

运行：

```powershell
go run .\cmd\event_producer
```

查看队列：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_queues name messages_ready messages_unacknowledged
```

预期：

```text
stage3.order.created.queue 1 0
```

## 6. 代码关键点解释

### ExchangeDeclare

```go
ch.ExchangeDeclare(name, kind, durable, autoDelete, internal, noWait, args)
```

本例：

```text
name: stage3.events.topic
kind: topic
durable: true
autoDelete: false
internal: false
```

### QueueDeclare

```go
ch.QueueDeclare(name, durable, autoDelete, exclusive, noWait, args)
```

业务队列通常：

```text
durable: true
autoDelete: false
exclusive: false
```

### QueueBind

```go
ch.QueueBind(queueName, routingKey, exchangeName, noWait, args)
```

它建立：

```text
exchange -- routing key --> queue
```

## 7. 声明要保持参数一致

RabbitMQ 的声明是幂等的前提是：

```text
同名资源的参数一致。
```

例如第一次创建：

```text
stage3.events.topic type=topic durable=true
```

第二次声明如果写成：

```text
type=direct
```

会报错。

解决方式：

- 改回一致参数。
- 或删除旧资源后重建。

删除 exchange：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl delete_exchange stage3.events.topic
```

删除 queue：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl delete_queue stage3.order.created.queue
```

## 8. 本节小结

你已经用 Go 完成：

- 声明 exchange。
- 声明 queue。
- 绑定 queue。
- 发布业务事件。

下一节开始实现 Work Queue。

