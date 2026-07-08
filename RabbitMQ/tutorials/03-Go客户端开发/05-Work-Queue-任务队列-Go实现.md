# 05. Work Queue：任务队列 Go 实现

## 1. 本节目标

实现任务队列模式：

```text
task_producer -> stage3.task.direct -- email.send --> stage3.email.task.queue -> task_worker
```

特点：

- producer 发送任务。
- 多个 worker 可以共同消费同一个队列。
- 每条任务通常只由一个 worker 处理。

## 2. 创建任务生产者

创建：

```text
cmd/task_producer/main.go
```

代码：

```go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"strings"
	"time"

	amqp "github.com/rabbitmq/amqp091-go"
)

const rabbitURL = "amqp://go_learner:go_learner_pwd@localhost:5672/"
const exchangeName = "stage3.task.direct"
const queueName = "stage3.email.task.queue"
const routingKey = "email.send"

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

	body := bodyFromArgs()
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
			Body:         []byte(body),
			Timestamp:    time.Now(),
		},
	)
	if err != nil {
		log.Fatalf("publish task: %v", err)
	}

	log.Printf("sent task: %s", body)
}

func declareTopology(ch *amqp.Channel) error {
	if err := ch.ExchangeDeclare(exchangeName, "direct", true, false, false, false, nil); err != nil {
		return fmt.Errorf("exchange declare: %w", err)
	}

	q, err := ch.QueueDeclare(queueName, true, false, false, false, nil)
	if err != nil {
		return fmt.Errorf("queue declare: %w", err)
	}

	if err := ch.QueueBind(q.Name, routingKey, exchangeName, false, nil); err != nil {
		return fmt.Errorf("queue bind: %w", err)
	}

	return nil
}

func bodyFromArgs() string {
	if len(os.Args) < 2 {
		return "send welcome email"
	}
	return strings.Join(os.Args[1:], " ")
}
```

运行：

```powershell
go run .\cmd\task_producer "send email to user-001"
go run .\cmd\task_producer "send email to user-002"
go run .\cmd\task_producer "send email to user-003"
```

查看队列：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_queues name messages_ready consumers
```

预期：

```text
stage3.email.task.queue 3 0
```

## 3. 创建任务消费者

创建：

```text
cmd/task_worker/main.go
```

代码：

```go
package main

import (
	"fmt"
	"log"
	"strings"
	"time"

	amqp "github.com/rabbitmq/amqp091-go"
)

const rabbitURL = "amqp://go_learner:go_learner_pwd@localhost:5672/"
const exchangeName = "stage3.task.direct"
const queueName = "stage3.email.task.queue"
const routingKey = "email.send"

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

	if err := ch.Qos(1, 0, false); err != nil {
		log.Fatalf("set qos: %v", err)
	}

	msgs, err := ch.Consume(queueName, "", false, false, false, false, nil)
	if err != nil {
		log.Fatalf("register consumer: %v", err)
	}

	log.Println("worker started. press Ctrl+C to exit")

	for d := range msgs {
		body := string(d.Body)
		log.Printf("received task: %s", body)
		simulateWork(body)
		if err := d.Ack(false); err != nil {
			log.Printf("ack failed: %v", err)
			continue
		}
		log.Printf("done: %s", body)
	}
}

func declareTopology(ch *amqp.Channel) error {
	if err := ch.ExchangeDeclare(exchangeName, "direct", true, false, false, false, nil); err != nil {
		return fmt.Errorf("exchange declare: %w", err)
	}

	q, err := ch.QueueDeclare(queueName, true, false, false, false, nil)
	if err != nil {
		return fmt.Errorf("queue declare: %w", err)
	}

	if err := ch.QueueBind(q.Name, routingKey, exchangeName, false, nil); err != nil {
		return fmt.Errorf("queue bind: %w", err)
	}

	return nil
}

func simulateWork(body string) {
	seconds := strings.Count(body, ".") + 1
	time.Sleep(time.Duration(seconds) * time.Second)
}
```

运行一个 worker：

```powershell
go run .\cmd\task_worker
```

再打开第二个 PowerShell，运行第二个 worker：

```powershell
cd C:\Users\MSI\Desktop\RabbitMQ\demos\stage3-go-client
go run .\cmd\task_worker
```

然后继续发布任务：

```powershell
go run .\cmd\task_producer "task A..."
go run .\cmd\task_producer "task B."
go run .\cmd\task_producer "task C....."
```

你会看到多个 worker 分担任务。

## 4. Qos 的作用

```go
ch.Qos(1, 0, false)
```

这里的 `prefetchCount=1` 表示：

```text
RabbitMQ 不要一次性给这个消费者太多未确认消息。
```

一个 worker 处理完并 ack 后，再接收下一条。

prefetch 会在第 4 阶段深入学习。

## 5. manual ack 的作用

消费者注册时：

```go
ch.Consume(queueName, "", false, false, false, false, nil)
```

第三个参数 `autoAck=false`，表示手动 ack。

处理完成后：

```go
d.Ack(false)
```

这表示：

```text
业务处理成功后，再告诉 RabbitMQ 可以删除这条消息。
```

可靠性细节后续阶段会深入。

## 6. Work Queue 的关键点

你要记住：

- 多个 worker 消费同一个 queue。
- 一条任务通常只由一个 worker 处理。
- 增加 worker 可以提升吞吐。
- 任务消费要考虑 ack、prefetch、失败重试和幂等。

## 7. 常见问题

### worker 没收到任务

检查队列名是否一致：

```text
stage3.email.task.queue
```

检查 binding：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_bindings
```

### 任务一直在 Unacked

说明消息已经发给 consumer，但没有 ack。

检查 worker 是否卡住，或者是否忘记：

```go
d.Ack(false)
```

## 8. 本节小结

你已经用 Go 实现 Work Queue。

下一节实现 fanout 发布订阅。

