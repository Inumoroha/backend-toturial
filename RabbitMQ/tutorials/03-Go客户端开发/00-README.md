# 第 3 阶段：Go 客户端开发

> 本阶段目标：使用 Go 和 `github.com/rabbitmq/amqp091-go` 写出 RabbitMQ 的基础生产者和消费者，并完成 Hello World、Work Queue、Publish/Subscribe、Routing、Topics 等核心模式。

## 学习顺序

请按下面顺序学习：

1. [01-本阶段目标与项目准备.md](./01-本阶段目标与项目准备.md)
2. [02-Go连接RabbitMQ-Connection与Channel.md](./02-Go连接RabbitMQ-Connection与Channel.md)
3. [03-Hello-World-默认交换机生产者消费者.md](./03-Hello-World-默认交换机生产者消费者.md)
4. [04-声明Exchange-Queue-Binding.md](./04-声明Exchange-Queue-Binding.md)
5. [05-Work-Queue-任务队列-Go实现.md](./05-Work-Queue-任务队列-Go实现.md)
6. [06-Publish-Subscribe-Fanout-Go实现.md](./06-Publish-Subscribe-Fanout-Go实现.md)
7. [07-Routing-Direct-Go实现.md](./07-Routing-Direct-Go实现.md)
8. [08-Topics-Topic-Go实现.md](./08-Topics-Topic-Go实现.md)
9. [09-消息结构-JSON-Properties-Headers.md](./09-消息结构-JSON-Properties-Headers.md)
10. [10-消费者优雅退出与基础错误处理.md](./10-消费者优雅退出与基础错误处理.md)
11. [11-Go客户端常见问题排查.md](./11-Go客户端常见问题排查.md)
12. [12-第3阶段练习与自测.md](./12-第3阶段练习与自测.md)

## 建议学习时间

建议用 7 到 10 天完成：

- 第 1 天：准备 Go 项目，安装 `amqp091-go`。
- 第 2 天：理解 Connection 和 Channel。
- 第 3 天：完成 Hello World。
- 第 4 天：练习声明 exchange、queue、binding。
- 第 5 天：完成 Work Queue。
- 第 6 天：完成 Fanout 发布订阅。
- 第 7 天：完成 Direct Routing。
- 第 8 天：完成 Topic 路由。
- 第 9 到 10 天：整理消息结构、优雅退出、排查清单，完成自测。

## 本阶段你要掌握什么

完成本阶段后，你应该能做到：

- 使用 Go 连接 RabbitMQ。
- 理解 Connection 和 Channel 的区别。
- 用 Go 声明 queue、exchange、binding。
- 用 Go 发布消息。
- 用 Go 消费消息。
- 实现默认交换机直投。
- 实现 Work Queue。
- 实现 fanout 发布订阅。
- 实现 direct routing。
- 实现 topic routing。
- 使用 JSON 作为消息体。
- 使用 message properties 和 headers。
- 处理基础错误和消费者退出。

## 本阶段暂时不深入什么

这些内容会放到第 4 阶段：

- publisher confirm
- mandatory publish
- manual ack 的完整可靠性设计
- nack、reject、requeue
- prefetch 调优
- 消费者断线重连
- 重试队列
- 死信队列
- 消息持久化的完整链路

本阶段可以使用 manual ack，但重点是先把 Go 客户端基础模式跑通。

## 官方资料

- [RabbitMQ Go Tutorial: Hello World](https://www.rabbitmq.com/tutorials/tutorial-one-go)
- [RabbitMQ Go Tutorial: Work Queues](https://www.rabbitmq.com/tutorials/tutorial-two-go)
- [RabbitMQ Go Tutorial: Publish/Subscribe](https://www.rabbitmq.com/tutorials/tutorial-three-go)
- [RabbitMQ Go Tutorial: Routing](https://www.rabbitmq.com/tutorials/tutorial-four-go)
- [RabbitMQ Go Tutorial: Topics](https://www.rabbitmq.com/tutorials/tutorial-five-go)
- [rabbitmq/amqp091-go](https://github.com/rabbitmq/amqp091-go)
- [amqp091-go package docs](https://pkg.go.dev/github.com/rabbitmq/amqp091-go)
