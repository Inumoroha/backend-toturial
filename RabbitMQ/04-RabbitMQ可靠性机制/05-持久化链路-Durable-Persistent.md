# 05. 持久化链路：Durable 和 Persistent

## 1. 持久化解决什么问题

持久化关注的是：

```text
RabbitMQ 重启或节点故障时，资源定义和消息是否尽量保留下来。
```

学习 RabbitMQ 时，很多人会误以为：

```text
queue durable = 消息一定不丢
```

这是错误的。

可靠持久化至少要看三件事：

```text
durable exchange
durable queue
persistent message
```

生产端还要配合：

```text
publisher confirm
```

## 2. Durable Exchange

Durable exchange 表示：

```text
RabbitMQ 重启后，exchange 定义仍然存在。
```

Go 中：

```go
err := ch.ExchangeDeclare(
	"stage4.reliable.direct",
	"direct",
	true,
	false,
	false,
	false,
	nil,
)
```

第三个参数 `true` 就是 durable。

## 3. Durable Queue

Durable queue 表示：

```text
RabbitMQ 重启后，queue 定义仍然存在。
```

Go 中：

```go
q, err := ch.QueueDeclare(
	"stage4.reliable.queue",
	true,
	false,
	false,
	false,
	nil,
)
```

第二个参数 `true` 就是 durable。

## 4. Persistent Message

Persistent message 表示：

```text
消息本身要求持久化。
```

Go 中：

```go
amqp.Publishing{
	DeliveryMode: amqp.Persistent,
	Body:         body,
}
```

如果不设置，默认可能是非持久消息。

## 5. 三者要一起使用

推荐：

```text
durable exchange: true
durable queue: true
message delivery mode: persistent
```

示例：

```go
err = ch.ExchangeDeclare("stage4.reliable.direct", "direct", true, false, false, false, nil)
if err != nil {
	log.Fatalf("declare exchange: %v", err)
}

q, err := ch.QueueDeclare("stage4.reliable.queue", true, false, false, false, nil)
if err != nil {
	log.Fatalf("declare queue: %v", err)
}

err = ch.QueueBind(q.Name, "reliable.task", "stage4.reliable.direct", false, nil)
if err != nil {
	log.Fatalf("bind queue: %v", err)
}

err = ch.PublishWithContext(ctx, "stage4.reliable.direct", "reliable.task", false, false, amqp.Publishing{
	ContentType:  "application/json",
	DeliveryMode: amqp.Persistent,
	Body:         body,
})
```

## 6. 持久化不等于发布成功

即使你设置了：

```text
durable exchange
durable queue
persistent message
```

生产者仍然可能遇到：

- 网络断开。
- RabbitMQ 拒绝消息。
- 消息还没真正落盘，连接就断了。
- 发布超时。

所以生产端还需要：

```text
publisher confirm
```

下一节会重点学习。

## 7. 持久化不等于不会重复

持久化解决的是消息尽量不因 broker 重启而丢失。

它不解决：

- 消费者重复处理。
- 业务幂等。
- 下游处理失败。
- 消息乱序。

这些需要 ack、重试、幂等等机制配合。

## 8. 重启实验

创建 durable queue 并发布 persistent message 后：

```powershell
docker restart rabbitmq-dev
```

等待 RabbitMQ 启动完成：

```powershell
docker logs -f rabbitmq-dev
```

看到启动完成后，查看队列：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_queues name messages_ready messages_unacknowledged
```

如果消息仍在，说明持久化配置生效。

注意：学习环境中的 Docker 容器如果被删除，容器内数据也可能被删除。生产环境需要挂载数据卷。

## 9. Docker 数据卷提醒

学习阶段前面使用的命令没有显式挂载数据卷：

```powershell
docker run ... rabbitmq:4-management
```

容器删除后数据会丢。

如果你想保留 RabbitMQ 数据，可以使用 volume：

```powershell
docker volume create rabbitmq_data
docker run -d --hostname rabbitmq-dev --name rabbitmq-dev -p 5672:5672 -p 15672:15672 -v rabbitmq_data:/var/lib/rabbitmq -e RABBITMQ_DEFAULT_USER=go_learner -e RABBITMQ_DEFAULT_PASS=go_learner_pwd rabbitmq:4-management
```

注意：如果你已经有同名容器，需要先停止并删除旧容器。

## 10. 本节小结

你要记住：

- durable exchange 保存 exchange 定义。
- durable queue 保存 queue 定义。
- persistent message 表示消息需要持久化。
- 三者要配合使用。
- 持久化还需要 publisher confirm 才能让生产者知道 broker 已确认。
- Docker 容器删除会影响学习环境数据。

下一节学习 publisher confirm。

