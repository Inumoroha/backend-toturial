# RabbitMQ Go 后端工程师系统学习路线图

> 目标：面向 Go 后端开发，系统掌握 RabbitMQ 的核心概念、Go 客户端开发、可靠消息投递、失败重试、监控运维和真实业务落地能力。

## 0. 学习定位

RabbitMQ 不是单纯的“消息队列工具”，而是后端系统中的异步通信基础设施。作为 Go 后端工程师，你最终需要能回答并解决这些问题：

- 为什么要引入消息队列，而不是直接同步调用？
- 如何用 RabbitMQ 解耦服务、削峰填谷、异步处理任务？
- 如何保证消息尽量不丢、不重复处理、失败可恢复？
- 如何设计重试、死信队列、延迟任务、消费者并发和幂等？
- RabbitMQ、Kafka、Redis Stream、数据库任务表各自适合什么场景？
- 线上 RabbitMQ 出现消息堆积、消费变慢、连接爆炸、磁盘告警时如何排查？

Go 实战建议使用 RabbitMQ 官方团队维护的 AMQP 0-9-1 Go 客户端：

```bash
go get github.com/rabbitmq/amqp091-go
```

## 阶段教程目录

- [第 0 阶段：学习定位与基础认知](./tutorials/00-学习定位与基础认知/00-README.md)
- [第 1 阶段：RabbitMQ 安装、管理后台与基础模型](./tutorials/01-RabbitMQ安装与基础模型/00-README.md)
- [第 2 阶段：RabbitMQ 核心模型深入](./tutorials/02-RabbitMQ核心模型深入/00-README.md)
- [第 3 阶段：Go 客户端开发](./tutorials/03-Go客户端开发/00-README.md)
- [第 4 阶段：RabbitMQ 可靠性机制](./tutorials/04-RabbitMQ可靠性机制/00-README.md)
- [第 5 阶段：业务模式落地](./tutorials/05-业务模式落地/00-README.md)
- [第 6 阶段：生产运维](./tutorials/06-生产运维/00-README.md)
- [第 7 阶段：架构取舍与高级能力](./tutorials/07-架构取舍与高级能力/00-README.md)
- [第 8 阶段：综合实战与作品集](./tutorials/08-综合实战与作品集/00-README.md)

## 补充专题文档

- [RabbitMQ 后续进阶学习指南](./RabbitMQ_Advanced_Learning_Guide.md)
- [RabbitMQ 常见面试问题与参考答案](./RabbitMQ_Interview_QA.md)

## 1. 前置能力

在正式深入 RabbitMQ 前，建议先具备以下基础：

### Go 基础

- `context.Context` 的超时、取消和传递
- goroutine、channel、sync、errgroup
- HTTP API、JSON 编解码、配置管理
- 日志、错误处理、优雅退出
- 单元测试、集成测试、Docker Compose

### 后端基础

- TCP 长连接和短连接成本
- 事务、幂等、重试、超时、限流
- 服务拆分、异步任务、事件驱动
- 基础 Linux、Docker、Prometheus/Grafana 概念

## 2. 总体学习阶段

建议周期：8 到 12 周。每天 1.5 到 2.5 小时即可推进。

| 阶段 | 主题 | 目标 |
| --- | --- | --- |
| 第 1 阶段 | 消息队列基础 | 理解 MQ 解决什么问题 |
| 第 2 阶段 | RabbitMQ 核心模型 | 掌握 Exchange、Queue、Binding、Routing Key |
| 第 3 阶段 | Go 客户端开发 | 能写生产者、消费者、工作队列和发布订阅 |
| 第 4 阶段 | 可靠性机制 | 掌握 ack、prefetch、confirm、持久化、重试、DLX |
| 第 5 阶段 | 业务模式落地 | 订单、通知、任务调度、异步处理 |
| 第 6 阶段 | 生产运维 | 监控、告警、容量、权限、TLS、集群 |
| 第 7 阶段 | 架构取舍 | 能判断 RabbitMQ 适不适合某个场景 |
| 第 8 阶段 | 综合实战与作品集 | 做出可运行、可测试、可展示的 Go + RabbitMQ 项目 |

## 3. 第 1 阶段：消息队列基础

### 学习重点

- 同步调用 vs 异步消息
- 队列、生产者、消费者、Broker
- 削峰填谷、异步解耦、广播通知、任务分发
- 至少一次、至多一次、尽量一次的语义差异
- 消息丢失、消息重复、消息乱序、消息堆积

### 你要能讲清楚

- 用户下单后，为什么不应该同步完成发短信、扣库存、生成报表等所有动作？
- 消息队列能解决什么，不能解决什么？
- 为什么“消息不重复”不能只依赖 RabbitMQ，而要业务侧做幂等？

### 练习

- 画出一个“用户注册后发送欢迎邮件”的异步架构图。
- 写一段文字说明：如果邮件服务挂了，注册接口为什么仍然可以成功？

## 4. 第 2 阶段：RabbitMQ 核心模型

### 必学概念

- Producer：生产者
- Consumer：消费者
- Broker：RabbitMQ 服务端
- Connection：TCP 连接，应该长期复用
- Channel：AMQP 信道，基于连接创建，适合作为轻量通信通道
- Exchange：交换机，负责路由消息
- Queue：队列，负责存储消息
- Binding：绑定关系
- Routing Key：路由键
- Virtual Host：虚拟主机，用于隔离环境或业务

### Exchange 类型

| 类型 | 作用 | 常见场景 |
| --- | --- | --- |
| direct | 精确匹配 routing key | 订单状态变更、指定任务类型 |
| fanout | 广播到所有绑定队列 | 多系统监听同一事件 |
| topic | 通配符路由 | `order.*`、`user.created` 等事件路由 |
| headers | 基于 header 匹配 | 少用，了解即可 |

### 练习

1. 使用 Docker 启动 RabbitMQ Management。
2. 在管理后台手动创建 exchange、queue、binding。
3. 分别测试 direct、fanout、topic 三种路由方式。

示例 Docker 命令：

```bash
docker run -d --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  rabbitmq:management
```

管理后台默认地址：

```text
http://localhost:15672
```

本地默认账号通常是：

```text
guest / guest
```

## 5. 第 3 阶段：Go + RabbitMQ 入门实战

### 学习重点

- 使用 `github.com/rabbitmq/amqp091-go`
- 建立连接和 channel
- 声明队列、交换机、绑定关系
- 发布消息
- 消费消息
- 手动 ack
- 使用 `context.WithTimeout` 控制发布超时
- 优雅关闭生产者和消费者

### 推荐代码结构

```text
rabbitmq-demo/
  cmd/
    producer/
      main.go
    consumer/
      main.go
  internal/
    mq/
      connection.go
      publisher.go
      consumer.go
  docker-compose.yml
  go.mod
```

### 第一批必须完成的小实验

- Hello World：一个 producer，一个 consumer。
- Work Queue：多个 worker 共同消费一个任务队列。
- Publish/Subscribe：一条消息广播给多个服务。
- Routing：根据 routing key 分发消息。
- Topic：用 `order.created`、`order.paid`、`user.created` 做主题路由。
- RPC：了解 RabbitMQ RPC 模式，但不要默认把它作为首选架构。

### Go 编码习惯

- 连接应长期复用，不要每发一条消息就新建连接。
- 发布和消费最好使用不同连接，避免发布端流控影响消费者 ack。
- channel 出错后通常需要重新创建。
- 消费者必须监听进程退出信号，关闭前尽量完成当前消息处理。
- 日志里记录 message id、correlation id、routing key、retry count。

## 6. 第 4 阶段：可靠性机制

这是 RabbitMQ 学习的核心阶段。

### 生产端可靠性

必须理解：

- durable exchange
- durable queue
- persistent message
- publisher confirm
- mandatory publish 与 unroutable message
- 发布失败后的重试策略

注意：只声明 durable queue 不代表消息一定不丢。队列、交换机、消息持久化和 publisher confirm 要配合使用。

### 消费端可靠性

必须理解：

- auto ack 和 manual ack 的区别
- ack、nack、reject
- requeue 的风险
- prefetch 如何影响吞吐和公平分发
- 消费者处理失败时如何进入重试或死信

### 重点心法

- RabbitMQ 可以做到可靠投递链路的一部分，但业务最终一致性要靠整体设计。
- 至少一次投递意味着消费者必须能处理重复消息。
- 不要无限 requeue，否则容易形成失败消息反复打爆消费者的循环。
- 长时间处理的任务要关注 consumer timeout、心跳和进程优雅退出。

## 7. 第 4 阶段补充：重试、死信和延迟任务

### 必学机制

- DLX：Dead Letter Exchange，死信交换机
- DLQ：Dead Letter Queue，死信队列
- TTL：消息或队列过期时间
- 最大重试次数
- retry count 记录方式
- poison message：毒消息

### 推荐重试设计

```text
业务队列 -> 消费失败 -> retry queue with TTL -> 过期后回到业务队列
业务队列 -> 超过最大重试次数 -> dead letter queue
```

### 实战任务

实现一个“订单超时取消”小系统：

- 用户创建订单后发送 `order.created` 消息。
- 延迟 15 分钟检查订单是否已支付。
- 如果未支付，发送 `order.cancelled` 消息。
- 如果处理失败，最多重试 3 次。
- 超过重试次数进入死信队列。

## 8. 第 5 阶段补充：Go 工程化封装

### 你需要封装的能力

- RabbitMQ 配置结构体
- 连接管理
- publisher confirm 支持
- consumer handler 抽象
- 统一日志
- 消息序列化和反序列化
- 重试次数读取和写入
- 幂等键生成
- 优雅关闭
- 健康检查

### 不建议过早封装

初学阶段不要一上来就写复杂 MQ SDK。推荐先写 3 到 5 个业务场景，再抽取真正重复的部分。

### 推荐抽象

```go
type Publisher interface {
    Publish(ctx context.Context, exchange, routingKey string, body []byte, opts PublishOptions) error
}

type Handler interface {
    Handle(ctx context.Context, msg Message) error
}
```

## 9. 第 5 阶段：业务架构模式落地

### 常见落地场景

- 用户注册后异步发邮件、短信、站内信
- 订单创建后异步扣库存、生成物流单、通知商家
- 文件上传后异步转码、压缩、审核
- 支付成功后广播给订单、积分、优惠券、风控服务
- 后台批量任务分发给多个 worker

### 必学业务模式

- Transactional Outbox：事务性发件箱
- Idempotent Consumer：幂等消费者
- Saga：长事务拆分
- Event Driven Architecture：事件驱动架构
- CQRS 入门

### 重点问题

数据库事务成功了，但消息发送失败怎么办？

推荐学习 Transactional Outbox：

```text
本地事务内同时写业务表和 outbox 表
后台任务扫描 outbox 表并投递 RabbitMQ
投递成功后标记 outbox 消息已发送
消费者通过 message_id 或业务唯一键做幂等
```

## 10. 第 6 阶段：监控、运维与线上排查

### RabbitMQ 管理能力

- Management UI
- `rabbitmqctl`
- `rabbitmq-diagnostics`
- vhost、user、permission
- policy
- shovel、federation 了解即可

### 需要监控的核心指标

- 队列 ready 消息数
- 队列 unacked 消息数
- 发布速率
- 消费速率
- ack 速率
- consumer 数量
- connection/channel 数量
- 内存使用
- 磁盘剩余
- file descriptors
- node health

### 常见故障排查

| 现象 | 可能原因 |
| --- | --- |
| ready 消息持续增长 | 消费者数量不足、消费逻辑变慢、下游服务慢 |
| unacked 很高 | 消费者拿到消息但没有 ack，可能卡住或 prefetch 太大 |
| 连接数暴涨 | 应用频繁创建短连接 |
| 发布变慢 | broker flow control、磁盘或内存压力 |
| 消息重复 | 消费者 ack 前崩溃、重试、网络断开 |
| 消息丢失 | 未持久化、未 confirm、auto ack、错误 requeue 策略 |

## 11. 第 7 阶段：架构取舍与高级能力

### 队列类型

- Classic Queue：经典队列，适合基础场景。
- Quorum Queue：基于 Raft 的复制队列，更适合高可用和数据安全场景。
- Stream：面向日志式、可重复读取、高吞吐事件流场景。

### 需要掌握

- 单节点和集群的区别
- 元数据和消息数据的复制差异
- quorum queue 的适用场景
- stream 和普通 queue 的差异
- 网络分区影响
- policy 如何统一管理队列参数
- 生产环境为什么不推荐随意使用临时队列和短连接

### 不必一开始深挖

- Erlang VM 内部实现
- RabbitMQ 插件开发
- 大规模跨地域集群
- 自定义协议扩展

## 12. RabbitMQ vs Kafka vs Redis Stream

| 技术 | 更适合 | 不太适合 |
| --- | --- | --- |
| RabbitMQ | 业务消息、任务队列、复杂路由、低延迟异步处理 | 大规模日志留存和长时间回放 |
| Kafka | 高吞吐日志、事件流、数据管道、可回放事件 | 复杂 per-message 路由、轻量任务队列 |
| Redis Stream | 轻量消息流、已有 Redis 技术栈、小规模异步消费 | 强可靠、复杂运维隔离、大型消息系统 |
| 数据库任务表 | 简单后台任务、强事务一致性 | 高并发消息分发、复杂路由 |

## 13. 12 周学习计划

### 第 1 周：MQ 基础和 RabbitMQ 安装

- 理解消息队列的价值和代价。
- 使用 Docker 启动 RabbitMQ。
- 熟悉 Management UI。
- 手动创建 exchange、queue、binding。

产出：一篇学习笔记，解释 Producer、Consumer、Exchange、Queue、Binding。

### 第 2 周：RabbitMQ 路由模型

- direct exchange
- fanout exchange
- topic exchange
- routing key
- binding key

产出：画出三种 exchange 的消息流向图。

### 第 3 周：Go 入门代码

- 使用 `amqp091-go` 写 producer。
- 使用 `amqp091-go` 写 consumer。
- 完成 Hello World、Work Queue。
- 用 `context.Context` 控制发布超时。

产出：一个可运行的 Go demo 仓库。

### 第 4 周：发布订阅和主题路由

- 实现日志广播。
- 实现订单事件 topic 路由。
- 多消费者组分别处理不同业务。

产出：`order.created`、`order.paid`、`order.cancelled` 三类事件 demo。

### 第 5 周：ack、prefetch 和消费者并发

- manual ack
- nack/reject
- prefetch
- 多 worker 并发消费
- 优雅退出

产出：一个支持并发 worker、失败 nack、成功 ack 的消费者。

### 第 6 周：publisher confirm 和持久化

- durable queue
- persistent message
- publisher confirm
- mandatory publish
- unroutable message 处理

产出：一个可靠发布器，能感知发布成功、失败、不可路由。

### 第 7 周：重试与死信队列

- DLX
- TTL
- retry queue
- max retry
- poison message

产出：失败重试 3 次后进入 DLQ 的完整 demo。

### 第 8 周：业务项目一：通知系统

实现一个通知服务：

- HTTP API 接收发送邮件请求。
- API 写入消息队列。
- worker 异步消费。
- 失败自动重试。
- 最终失败进入死信队列。
- 消费者保证幂等。

产出：可运行的 `notification-service`。

### 第 9 周：业务项目二：订单事件系统

实现一个简化订单系统：

- 创建订单
- 支付订单
- 发布订单事件
- 库存服务消费事件
- 积分服务消费事件
- 风控服务消费事件

产出：基于 topic exchange 的事件驱动 demo。

### 第 10 周：Outbox 和最终一致性

- 学习本地事务 + outbox 表。
- 后台投递 outbox 消息。
- 消费者幂等处理。
- 模拟数据库成功但 MQ 暂时不可用。

产出：一个解决“数据库事务成功但消息发送失败”的 demo。

### 第 11 周：监控和排查

- Management UI 指标观察。
- Prometheus/Grafana 了解。
- 模拟消息堆积。
- 模拟消费者崩溃。
- 模拟网络断开。

产出：一份 RabbitMQ 故障排查清单。

### 第 12 周：高可用和架构总结

- 学习 quorum queue。
- 了解 cluster。
- 了解 stream。
- 对比 RabbitMQ、Kafka、Redis Stream。
- 整理面试题和项目亮点。

产出：一篇“我会如何在 Go 项目中使用 RabbitMQ”的总结文档。

## 14. 综合实战项目

### 项目名称：Go 电商异步订单系统

### 技术栈

- Go
- Gin 或 Chi
- PostgreSQL 或 MySQL
- RabbitMQ
- Docker Compose
- Prometheus/Grafana 可选

### 服务拆分

```text
api-service
order-service
inventory-service
notification-service
payment-mock-service
worker-service
```

### 核心流程

1. 用户创建订单。
2. order-service 写入订单表和 outbox 表。
3. outbox worker 发布 `order.created`。
4. inventory-service 扣减库存。
5. payment-mock-service 模拟支付成功。
6. order-service 发布 `order.paid`。
7. notification-service 发送通知。
8. 任意消费失败进入重试队列。
9. 超过最大重试次数进入死信队列。
10. 所有消费者通过业务唯一键保证幂等。

### 你要体现的工程能力

- 清晰的 exchange/queue 命名规范
- 消息 schema 版本号
- correlation id 链路追踪
- publisher confirm
- manual ack
- prefetch 控制
- retry + DLQ
- outbox
- graceful shutdown
- 指标监控
- 集成测试

## 15. 面试检查清单

学习完成后，你应该能回答：

- RabbitMQ 的 exchange、queue、binding 是什么关系？
- direct、fanout、topic exchange 有什么区别？
- Connection 和 Channel 有什么区别？
- 为什么不应该频繁创建连接？
- auto ack 有什么风险？
- prefetch 过大或过小分别会怎样？
- RabbitMQ 如何保证消息尽量不丢？
- publisher confirm 解决什么问题？
- durable queue 和 persistent message 有什么区别？
- 什么是 DLX？哪些情况会产生死信？
- 如何设计一个延迟重试队列？
- 如何避免消息重复消费造成业务错误？
- 消息积压如何排查？
- RabbitMQ 和 Kafka 怎么选？
- 数据库事务成功但消息发送失败怎么办？
- 什么是 quorum queue？适合什么场景？
- Go 客户端断线后如何恢复？

## 16. 推荐学习资料

优先看官方文档和官方教程：

- [RabbitMQ Go Tutorial: Hello World](https://www.rabbitmq.com/tutorials/tutorial-one-go)
- [RabbitMQ Go Tutorial: Publish/Subscribe](https://www.rabbitmq.com/tutorials/tutorial-three-go)
- [RabbitMQ Go Tutorial: RPC](https://www.rabbitmq.com/tutorials/tutorial-six-go)
- [RabbitMQ Consumer Acknowledgements and Publisher Confirms](https://www.rabbitmq.com/docs/confirms)
- [RabbitMQ Consumer Prefetch](https://www.rabbitmq.com/docs/consumer-prefetch)
- [RabbitMQ Dead Letter Exchanges](https://www.rabbitmq.com/docs/dlx)
- [RabbitMQ Time-To-Live and Expiration](https://www.rabbitmq.com/docs/ttl)
- [RabbitMQ Reliability Guide](https://www.rabbitmq.com/docs/reliability)
- [RabbitMQ Production Deployment Guidelines](https://www.rabbitmq.com/docs/production-checklist)
- [RabbitMQ Monitoring](https://www.rabbitmq.com/docs/monitoring)
- [RabbitMQ Quorum Queues](https://www.rabbitmq.com/docs/quorum-queues)
- [RabbitMQ Streams](https://www.rabbitmq.com/docs/streams)
- [rabbitmq/amqp091-go](https://github.com/rabbitmq/amqp091-go)

## 17. 学习节奏建议

每天按这个比例推进：

- 30%：读文档和画图。
- 50%：写 Go demo。
- 20%：记录问题、总结踩坑、补测试。

每学完一个机制，都问自己三个问题：

1. 它解决什么问题？
2. 它带来什么新风险？
3. 在 Go 项目里我要如何封装和测试它？

## 18. 最终目标

当你完成这条路线后，你不只是“会用 RabbitMQ 发消息”，而是应该能独立设计并落地一套 Go 后端异步消息方案：

- 能选型。
- 能编码。
- 能保证可靠性。
- 能处理失败。
- 能监控和排查。
- 能把 RabbitMQ 放进真实业务架构里。
