# 03. Hello World：默认交换机生产者消费者

## 1. 本节目标

本节使用 Go 完成最小闭环：

```text
producer -> 默认 exchange -> stage3.hello.queue -> consumer
```

你会写两个程序：

```text
cmd/hello_producer/main.go
cmd/hello_consumer/main.go
```

## 2. 默认交换机回顾

默认 exchange 的名称是空字符串：

```text
""
```

当你发布消息时：

```text
exchange: ""
routing key: stage3.hello.queue
```

如果存在同名队列 `stage3.hello.queue`，消息就会进入这个队列。

## 3. 创建生产者

创建文件：

```text
cmd/hello_producer/main.go
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
const queueName = "stage3.hello.queue"

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

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	body := "Hello RabbitMQ from Go"
	err = ch.PublishWithContext(
		ctx,
		"",
		q.Name,
		false,
		false,
		amqp.Publishing{
			ContentType:  "text/plain",
			DeliveryMode: amqp.Persistent,
			Body:         []byte(body),
		},
	)
	if err != nil {
		log.Fatalf("publish message: %v", err)
	}

	log.Printf("sent: %s", body)
}
```

运行：

```powershell
go run .\cmd\hello_producer
```

预期：

```text
sent: Hello RabbitMQ from Go
```

## 4. 查看队列

执行：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers
```

你应该看到：

```text
stage3.hello.queue 1 0 0
```

说明消息已经进入队列，等待消费。

## 5. 创建消费者

创建文件：

```text
cmd/hello_consumer/main.go
```

代码：

```go
package main

import (
	"log"

	amqp "github.com/rabbitmq/amqp091-go"
)

const rabbitURL = "amqp://go_learner:go_learner_pwd@localhost:5672/"
const queueName = "stage3.hello.queue"

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

	msgs, err := ch.Consume(
		q.Name,
		"stage3-hello-consumer",
		false,
		false,
		false,
		false,
		nil,
	)
	if err != nil {
		log.Fatalf("register consumer: %v", err)
	}

	log.Println("waiting for messages. press Ctrl+C to exit")

	forever := make(chan struct{})

	go func() {
		for d := range msgs {
			log.Printf("received: %s", string(d.Body))
			if err := d.Ack(false); err != nil {
				log.Printf("ack failed: %v", err)
			}
		}
	}()

	<-forever
}
```

运行：

```powershell
go run .\cmd\hello_consumer
```

预期：

```text
received: Hello RabbitMQ from Go
```

消费者会继续等待消息，按 `Ctrl+C` 退出。

## 6. 再次发布消息

保持 consumer 运行，打开另一个 PowerShell：

```powershell
cd C:\Users\MSI\Desktop\RabbitMQ\demos\stage3-go-client
go run .\cmd\hello_producer
```

你会看到 consumer 立刻收到消息。

## 7. 代码关键点解释

### 7.1 QueueDeclare

```go
q, err := ch.QueueDeclare(queueName, true, false, false, false, nil)
```

参数依次是：

```text
name
durable
autoDelete
exclusive
noWait
args
```

本例含义：

```text
创建 durable、非 auto-delete、非 exclusive 的业务队列。
```

### 7.2 PublishWithContext

```go
ch.PublishWithContext(ctx, "", q.Name, false, false, publishing)
```

参数中：

```text
exchange: ""
routing key: q.Name
```

表示使用默认 exchange 投递到同名队列。

### 7.3 Consume

```go
ch.Consume(q.Name, "stage3-hello-consumer", false, false, false, false, nil)
```

第三个参数 `autoAck` 为 `false`，表示需要手动 ack。

所以处理完成后调用：

```go
d.Ack(false)
```

## 8. 常见问题

### 队列参数不一致

如果你之前用不同参数创建过同名队列，可能出现：

```text
inequivalent arg 'durable'
```

解决：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl delete_queue stage3.hello.queue
```

然后重新运行 producer 或 consumer。

### 消费者没有收到消息

检查：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers
```

如果 `messages_ready` 为 0，说明队列里没有等待消费的消息。

如果 `consumers` 为 0，说明消费者没有注册成功。

## 9. 本节小结

你已经完成第一个 Go + RabbitMQ 闭环：

```text
Go producer -> RabbitMQ queue -> Go consumer
```

下一节会学习用 Go 声明显式 exchange、queue、binding。

