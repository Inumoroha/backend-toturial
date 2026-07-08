# 01 Kafka 的整体心智模型

Kafka 可以理解为一个分布式的、可持久化的事件日志系统。

它不像普通队列那样“消息被消费后就删除”。Kafka 的核心思想是：消息先追加写入日志，consumer 根据自己的 offset 决定读到哪里。

## Kafka 适合解决什么问题

在 Go 后端系统中，Kafka 常用于：

- 异步处理：下单成功后异步发通知、发优惠券。
- 削峰填谷：请求高峰先写 Kafka，消费者按能力慢慢处理。
- 系统解耦：订单服务不直接调用库存、积分、通知服务。
- 事件流：记录用户行为、订单状态变化、支付状态变化。
- 数据同步：把数据库变更同步到搜索、分析、缓存系统。

## Kafka 中的几个角色

### Producer

生产者，负责把消息写入 Kafka。

在 Go 项目里，producer 往往出现在：

- HTTP API 处理成功后发布事件。
- 定时任务扫描 outbox 表后发布事件。
- 日志采集程序写入行为日志。

### Broker

Kafka 服务器节点。一个 Kafka 集群由多个 broker 组成。

broker 负责：

- 接收 producer 写入。
- 存储 topic partition 数据。
- 响应 consumer 读取。
- 参与 leader 选举和副本复制。

### Topic

topic 是消息的逻辑分类。

例如：

- `order.created`
- `order.paid`
- `inventory.deducted`
- `user.registered`

topic 名要表达业务事件，而不是表达处理动作。推荐用“领域对象 + 过去式动作”，例如 `order.created`，表示“订单已经创建”。

### Partition

partition 是 topic 的物理分片。一个 topic 可以有多个 partition。

partition 的意义：

- 提高写入并行度。
- 提高读取并行度。
- 让数据分布到多个 broker。
- 在单个 partition 内保持顺序。

### Consumer

消费者，从 Kafka 读取消息并执行业务逻辑。

在 Go 项目里，consumer 往往是一个后台服务：

- 订单事件消费者。
- 库存扣减消费者。
- 通知消费者。
- 数据同步消费者。

### Consumer Group

consumer group 是 Kafka 非常关键的概念。

同一个 group 内的多个 consumer 会协作消费一个 topic。一个 partition 在同一时刻只能分配给 group 内的一个 consumer。

不同 group 之间互不影响，每个 group 都有自己的 offset。

## Kafka 和普通消息队列的关键区别

传统队列常见模型：

```text
生产者 -> 队列 -> 消费者
```

消息被消费后通常就从队列中删除。

Kafka 模型：

```text
producer -> topic partition log -> consumer group offset
```

消息是否还在 Kafka，取决于 topic 的保留策略，而不是某个 consumer 是否读过。

## 对 Go 后端最重要的理解

Kafka 不会自动保证你的业务一定“只成功一次”。

Kafka 能提供：

- 消息持久化。
- 消费位置记录。
- 分区内顺序。
- 副本容错。
- 至少一次消费能力。
- 特定场景下的事务语义。

但业务层仍然要处理：

- 重复消费。
- 消费失败。
- 外部接口重复调用。
- 数据库写入和消息发送的一致性。
- 下游服务不可用。

## 本节练习

1. 画出你理解的 Kafka 数据流：producer、topic、partition、consumer group。
2. 用自己的话解释：为什么 Kafka 的消息被消费后不会立即删除？
3. 思考：订单服务发布 `order.created` 后，哪些服务可能会消费这个事件？

