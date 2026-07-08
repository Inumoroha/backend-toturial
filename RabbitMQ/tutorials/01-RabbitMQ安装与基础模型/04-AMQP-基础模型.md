# 04. AMQP 基础模型

## 1. RabbitMQ 和 AMQP 的关系

RabbitMQ 支持多种协议，其中 Go 后端最常用的是 AMQP 0-9-1。

后续使用的 Go 客户端：

```text
github.com/rabbitmq/amqp091-go
```

就是 RabbitMQ 官方团队维护的 AMQP 0-9-1 Go 客户端。

第 1 阶段你不需要学习协议细节，只需要理解 RabbitMQ 的核心模型。

## 2. 最核心的四个角色

```text
Producer -> Exchange -> Queue -> Consumer
```

| 角色 | 中文 | 作用 |
| --- | --- | --- |
| Producer | 生产者 | 发布消息 |
| Exchange | 交换机 | 根据规则路由消息 |
| Queue | 队列 | 存储消息，等待消费 |
| Consumer | 消费者 | 从队列取消息并处理 |

一句话版：

```text
生产者发消息给交换机，交换机把消息路由到队列，消费者从队列消费。
```

## 3. Exchange：消息路由入口

Exchange 不负责长期保存消息，它负责决定消息应该去哪里。

生产者发布消息时，通常会指定：

- exchange name
- routing key
- message body
- message properties

RabbitMQ 根据 exchange 类型和 binding 规则决定消息进入哪些队列。

示例：

```text
exchange: order.events
routing key: order.created
body: {"order_id": 1001}
```

## 4. Queue：消息存储位置

Queue 是消息真正等待消费者处理的地方。

你可以把 Queue 想象成一个任务箱：

```text
任务消息进入队列
worker 从队列里取任务
处理成功后确认
```

一个 queue 可以有多个 consumer：

```text
task.queue
  -> worker-1
  -> worker-2
  -> worker-3
```

这种模式适合任务分发。

## 5. Binding：交换机和队列的关系

Binding 表示：

```text
某个 queue 对某个 exchange 的某类消息感兴趣
```

可以理解为订阅关系。

例如：

```text
queue: order-created-queue
binding exchange: order.events
binding key: order.created
```

含义是：

```text
order-created-queue 想接收 order.events 中 routing key 为 order.created 的消息
```

## 6. Routing Key：路由键

Routing Key 是生产者发布消息时附带的路由信息。

例如：

```text
order.created
order.paid
order.cancelled
user.registered
email.send
```

不同 exchange 类型会用不同方式解释 routing key。

例如 direct exchange 通常要求精确匹配：

```text
routing key = binding key
```

topic exchange 支持通配符：

```text
order.*
order.#
```

## 7. Broker、Connection、Channel

除了消息模型，还要理解客户端连接模型。

### Broker

Broker 就是 RabbitMQ 服务端。

本地 Docker 容器 `rabbitmq-dev` 里运行的 RabbitMQ，就是 broker。

### Connection

Connection 是应用程序和 RabbitMQ 之间的 TCP 连接。

```text
Go 应用 -> TCP Connection -> RabbitMQ
```

Connection 成本较高，应该复用。

### Channel

Channel 是 connection 上的轻量通信通道。

```text
Connection
  -> Channel 1 发布消息
  -> Channel 2 消费消息
  -> Channel 3 声明队列
```

后续 Go 代码里会频繁看到 `conn.Channel()`。

## 8. 默认 exchange

RabbitMQ 有一个特殊的默认 exchange，名称为空字符串：

```text
""
```

使用默认 exchange 时，routing key 通常等于 queue 名称，消息会被路由到同名队列。

例如：

```text
exchange: ""
routing key: hello.queue
```

如果存在名为 `hello.queue` 的队列，消息会进入这个队列。

初学者可以先知道它存在，但后续业务中更推荐显式声明自己的 exchange，例如：

```text
order.events
notification.direct
task.direct
```

## 9. 本节小结

你需要牢牢记住这张图：

```text
Producer
  -> Exchange
  -> Binding Rules
  -> Queue
  -> Consumer
```

并且能解释：

- Exchange 负责路由。
- Queue 负责存储。
- Binding 连接 exchange 和 queue。
- Routing Key 影响消息被路由到哪里。
- Consumer 从 queue 消费，而不是从 exchange 消费。

