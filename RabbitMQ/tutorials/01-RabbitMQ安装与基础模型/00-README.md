# 第 1 阶段：RabbitMQ 安装、管理后台与基础模型

> 本阶段目标：把 RabbitMQ 在本地跑起来，熟悉 Management UI，理解 Producer、Exchange、Queue、Binding、Routing Key、Consumer 之间的关系，并能手动完成一次消息发布和消费。

## 学习顺序

请按下面顺序学习：

1. [01-本阶段目标与环境检查.md](./01-本阶段目标与环境检查.md)
2. [02-使用-Docker-启动-RabbitMQ.md](./02-使用-Docker-启动-RabbitMQ.md)
3. [03-认识-RabbitMQ-管理后台.md](./03-认识-RabbitMQ-管理后台.md)
4. [04-AMQP-基础模型.md](./04-AMQP-基础模型.md)
5. [05-手动创建-Exchange-Queue-Binding.md](./05-手动创建-Exchange-Queue-Binding.md)
6. [06-手动发布和消费第一条消息.md](./06-手动发布和消费第一条消息.md)
7. [07-Direct-Fanout-Topic-交换机入门.md](./07-Direct-Fanout-Topic-交换机入门.md)
8. [08-vhost-用户-权限与命名规范入门.md](./08-vhost-用户-权限与命名规范入门.md)
9. [09-第1阶段练习与自测.md](./09-第1阶段练习与自测.md)

## 建议学习时间

建议用 3 到 5 天完成：

- 第 1 天：环境检查，使用 Docker 启动 RabbitMQ。
- 第 2 天：熟悉 Management UI。
- 第 3 天：理解 AMQP 基础模型，手动创建 exchange、queue、binding。
- 第 4 天：手动发布和消费消息，测试 direct、fanout、topic。
- 第 5 天：学习 vhost、用户、权限和命名规范，完成自测。

## 本阶段最终产出

完成本阶段后，你应该能做到：

- 本地启动 RabbitMQ。
- 打开 Management UI。
- 解释 `5672` 和 `15672` 两个端口的作用。
- 手动创建 exchange、queue、binding。
- 手动发布一条消息，并从队列中取出。
- 解释 direct、fanout、topic 三类 exchange 的基本区别。
- 使用 `rabbitmqctl` 查看 queues、exchanges、bindings。
- 知道 vhost、user、permission 的基础作用。

## 本阶段暂时不做什么

本阶段不写 Go 代码，也不深入可靠性机制。你只需要先把 RabbitMQ 的基础模型在脑子里建起来。

Go 代码会放到后续阶段：

```text
第 2 阶段：RabbitMQ 核心模型深入
第 3 阶段：Go 客户端开发
第 4 阶段：可靠性机制
```

## 官方资料

- [RabbitMQ Installing Guide](https://www.rabbitmq.com/docs/download)
- [RabbitMQ Management Plugin](https://www.rabbitmq.com/docs/management)
- [RabbitMQ AMQP 0-9-1 Model Explained](https://www.rabbitmq.com/tutorials/amqp-concepts)
- [RabbitMQ Exchanges](https://www.rabbitmq.com/docs/exchanges)
- [RabbitMQ Queues](https://www.rabbitmq.com/docs/queues)

