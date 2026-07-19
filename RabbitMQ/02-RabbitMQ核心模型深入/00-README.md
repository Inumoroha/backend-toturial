# 第 2 阶段：RabbitMQ 核心模型深入

> 本阶段目标：深入理解 RabbitMQ 的核心模型。你不仅要会创建 exchange、queue、binding，还要知道这些资源的参数含义、适用场景、常见坑，以及如何为真实 Go 后端项目设计消息拓扑。

## 学习顺序

请按下面顺序学习：

1. [01-本阶段目标与第1阶段复习.md](./01-本阶段目标与第1阶段复习.md)
2. [02-Exchange-深入-类型与声明参数.md](./02-Exchange-深入-类型与声明参数.md)
3. [03-Queue-深入-声明参数与队列类型.md](./03-Queue-深入-声明参数与队列类型.md)
4. [04-Binding-与-Routing-Key-深入.md](./04-Binding-与-Routing-Key-深入.md)
5. [05-默认交换机与队列直投.md](./05-默认交换机与队列直投.md)
6. [06-临时队列-独占队列-自动删除队列.md](./06-临时队列-独占队列-自动删除队列.md)
7. [07-Work-Queue-任务队列模式.md](./07-Work-Queue-任务队列模式.md)
8. [08-Publish-Subscribe-Routing-Topics-模式.md](./08-Publish-Subscribe-Routing-Topics-模式.md)
9. [09-消息拓扑设计与命名规范.md](./09-消息拓扑设计与命名规范.md)
10. [10-第2阶段练习与自测.md](./10-第2阶段练习与自测.md)

## 建议学习时间

建议用 5 到 7 天完成：

- 第 1 天：复习基础模型，深入 exchange。
- 第 2 天：深入 queue 参数和队列类型。
- 第 3 天：深入 binding、routing key、默认交换机。
- 第 4 天：学习临时队列、独占队列、自动删除队列。
- 第 5 天：理解 Work Queue、发布订阅、路由、主题模式。
- 第 6 到 7 天：完成拓扑设计练习和自测。

## 本阶段你要掌握什么

完成本阶段后，你应该能讲清楚：

- exchange 的 `type`、`durable`、`auto-delete`、`internal` 是什么。
- queue 的 `durable`、`exclusive`、`auto-delete` 是什么。
- Classic Queue、Quorum Queue、Stream 的基础区别。
- direct、fanout、topic 的真实路由规则。
- 默认交换机为什么能通过 queue name 直接投递。
- 临时队列、独占队列、自动删除队列适合什么场景。
- Work Queue 和 Publish/Subscribe 的区别。
- 一个业务事件应该如何设计 exchange、queue、binding、routing key。

## 本阶段暂时不深入什么

这些内容后续阶段再学：

- Go 客户端代码。
- manual ack、nack、reject。
- prefetch。
- publisher confirm。
- dead letter exchange。
- TTL、延迟重试。
- quorum queue 高可用细节。
- 生产环境监控调优。

## 官方资料

- [AMQP 0-9-1 Model Explained](https://www.rabbitmq.com/tutorials/amqp-concepts)
- [Exchanges](https://www.rabbitmq.com/docs/exchanges)
- [Queues](https://www.rabbitmq.com/docs/queues)
- [Consumers](https://www.rabbitmq.com/docs/consumers)
- [Work Queues Tutorial](https://www.rabbitmq.com/tutorials/tutorial-two-go)
- [Publish/Subscribe Tutorial](https://www.rabbitmq.com/tutorials/tutorial-three-go)
- [Routing Tutorial](https://www.rabbitmq.com/tutorials/tutorial-four-go)
- [Topics Tutorial](https://www.rabbitmq.com/tutorials/tutorial-five-go)

