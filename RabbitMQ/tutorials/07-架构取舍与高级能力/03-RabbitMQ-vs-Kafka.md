# 03. RabbitMQ vs Kafka

## 1. 一句话区别

RabbitMQ 更像：

```text
业务消息代理，擅长路由、任务队列、低延迟异步处理。
```

Kafka 更像：

```text
分布式事件日志，擅长高吞吐、事件保留、回放和数据管道。
```

## 2. 核心模型对比

### RabbitMQ

```text
Producer -> Exchange -> Queue -> Consumer
```

核心对象：

- exchange。
- queue。
- binding。
- routing key。
- ack。

RabbitMQ 强调：

```text
消息如何被路由到队列，消费者如何确认处理。
```

### Kafka

```text
Producer -> Topic -> Partition -> Consumer Group
```

核心对象：

- topic。
- partition。
- offset。
- consumer group。
- log retention。

Kafka 强调：

```text
事件以日志形式写入，消费者按 offset 读取。
```

## 3. 路由能力

RabbitMQ 路由能力很强：

- direct。
- fanout。
- topic。
- headers。

如果业务需要复杂 routing key 和多个队列绑定，RabbitMQ 很自然。

Kafka 的 topic 路由更简单，通常按 topic 和 partition 组织。

如果你需要大量复杂 per-message 路由，RabbitMQ 更顺手。

## 4. 回放能力

Kafka 的强项是事件保留和回放。

消费者可以基于 offset 重新读取历史数据。

适合：

- 日志采集。
- 用户行为流。
- 数据管道。
- 实时数仓。
- 多个消费组独立读取历史事件。

普通 RabbitMQ queue 中，消息被 ack 后就从队列移除，不适合长期回放。

## 5. 消费模型

RabbitMQ：

```text
消息进入 queue
consumer 消费后 ack
ack 后消息删除
```

Kafka：

```text
消息写入 topic partition
consumer group 读取并提交 offset
消息按保留策略留存
```

所以 Kafka 更适合：

```text
一个事件被多个系统在不同时间重复读取。
```

## 6. 吞吐和延迟

一般取舍：

```text
Kafka 更偏高吞吐事件流。
RabbitMQ 更偏低延迟业务消息和灵活路由。
```

但真实性能取决于：

- 部署规模。
- 消息大小。
- 持久化策略。
- 副本策略。
- 消费者处理能力。
- 网络和磁盘。

不要只靠“谁更快”选型。

## 7. 可靠性思路

RabbitMQ：

- publisher confirm。
- durable queue。
- persistent message。
- manual ack。
- retry/DLQ。

Kafka：

- producer ack。
- replication。
- offset commit。
- retention。
- consumer group rebalance。

两者都需要业务幂等。

不要以为 Kafka 或 RabbitMQ 能自动保证业务只执行一次。

## 8. 典型选择

选择 RabbitMQ：

- 订单事件通知多个业务服务。
- 异步发送邮件短信。
- 后台任务分发。
- 复杂 routing key。
- 低延迟业务消息。

选择 Kafka：

- 用户行为日志。
- 埋点数据流。
- 实时数据管道。
- 事件长期保留和回放。
- 多个消费组按 offset 独立消费。
- 高吞吐流式处理。

## 9. 可以同时使用吗

可以。

一个系统中常见组合：

```text
RabbitMQ -> 业务异步任务和服务解耦
Kafka    -> 数据流、日志、分析、回放
```

例如：

```text
order.created -> RabbitMQ 通知库存和通知服务
order.created -> Kafka 进入实时数仓和分析系统
```

## 10. 面试表达

可以这样说：

```text
RabbitMQ 更适合业务消息、任务队列和复杂路由。Kafka 更适合高吞吐事件流、日志保留和回放。如果订单支付成功后要通知积分、通知、风控服务，我会优先考虑 RabbitMQ。如果是用户行为日志进入实时数仓，我会优先考虑 Kafka。
```

## 11. 本节小结

RabbitMQ 和 Kafka 不是谁替代谁。

核心区别：

```text
RabbitMQ = broker/router/queue
Kafka = distributed log/event streaming
```

下一节对比 RabbitMQ 和 Redis Streams。

