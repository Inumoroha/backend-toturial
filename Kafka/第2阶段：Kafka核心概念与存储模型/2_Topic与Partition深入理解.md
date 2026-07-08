# 2. Topic 与 Partition 深入理解

本节目标：深入理解 topic 和 partition 的关系，知道 partition 为什么决定 Kafka 的并行能力、顺序边界和扩展方式。

第 1 阶段我们已经会创建 topic，也知道一个 topic 可以有多个 partition。本节要进一步回答几个更工程化的问题：

```text
为什么 topic 要拆 partition？
partition 数量是不是越多越好？
为什么同一个 consumer group 的并行度受 partition 限制？
为什么 Kafka 只能保证 partition 内有序？
Go 后端项目里应该怎样选择 message key？
```

---

## 一、Topic 是逻辑分类

Topic 是 Kafka 中消息的逻辑分类。

例如：

```text
order.created
payment.succeeded
inventory.deducted
user.registered
```

你可以把 topic 理解成一类事件的集合。

在 Go 后端项目中，topic 通常会出现在：

- producer 发布事件时。
- consumer 订阅事件时。
- retry topic 和 DLQ 命名时。
- 日志字段中。
- Prometheus 指标标签中。
- 告警规则中。

---

## 二、Partition 是 Topic 的物理分片

一个 topic 可以拆成多个 partition。

例如：

```text
topic: order.created

partition-0:
  offset 0
  offset 1
  offset 2

partition-1:
  offset 0
  offset 1

partition-2:
  offset 0
  offset 1
```

partition 不是业务分类，而是 Kafka 为了存储和并行处理做的物理分片。

不要把 partition 当成：

```text
华东订单 partition
华北订单 partition
VIP 订单 partition
普通订单 partition
```

这种业务分类通常不应该通过 partition 名义表达，因为 partition 没有业务名称，只有编号。

---

## 三、为什么需要 Partition

### 1. 提高写入吞吐

如果一个 topic 只有一个 partition，那么这个 topic 的写入主要落到一个 partition leader 上。

多个 partition 可以分布在多个 broker 上，producer 写入可以并行。

例如：

```text
order.created partition-0 -> broker-1
order.created partition-1 -> broker-2
order.created partition-2 -> broker-3
```

这样写入压力可以分散。

### 2. 提高消费并行度

同一个 consumer group 中，一个 partition 同一时刻只能分配给一个 consumer。

如果 topic 有 3 个 partition：

```text
consumer-A -> partition-0
consumer-B -> partition-1
consumer-C -> partition-2
```

最多 3 个 consumer 并行消费。

如果启动第 4 个 consumer：

```text
consumer-D -> 空闲
```

所以：

```text
同一个 group 的最大并行度 <= partition 数量
```

### 3. 分散存储压力

不同 partition 可以存放在不同 broker 的磁盘上。

这能避免一个 topic 的所有数据都压在一台机器上。

---

## 四、Partition 与顺序性

Kafka 保证：

```text
同一个 partition 内消息有序
```

Kafka 不保证：

```text
一个多 partition topic 的全局顺序
```

例如：

```text
partition-0:
  offset 0: order-1 created
  offset 1: order-1 paid

partition-1:
  offset 0: order-2 created
  offset 1: order-2 paid
```

partition 内 offset 是递增的，但不同 partition 之间没有统一顺序。

如果你希望同一个订单的事件按顺序处理，就应该让同一个订单的事件进入同一个 partition。

常用做法：

```text
message key = order_id
```

---

## 五、Message Key 如何影响 Partition

Producer 发送消息时可以带 key。

Kafka 客户端通常会根据 key 计算目标 partition。

直觉上可以理解为：

```text
partition = hash(key) % partition_count
```

实际不同客户端和版本的分区算法可能有细节差异，但核心思想是：

```text
相同 key 会稳定地进入同一个 partition
```

这让我们可以做到：

- 同一个订单事件有序。
- 同一个用户事件有序。
- 同一个商品库存事件有序。

---

## 六、如何选择 Key

常见选择：

| 业务事件 | 推荐 key | 原因 |
| --- | --- | --- |
| `order.created` | `order_id` | 同一订单事件要有序 |
| `payment.succeeded` | `order_id` 或 `payment_id` | 看后续按订单还是按支付单处理 |
| `inventory.deducted` | `sku_id` 或 `order_id` | 看是否更关注商品库存顺序 |
| `user.registered` | `user_id` | 同一用户事件要有序 |
| 行为日志 | `user_id` 或空 key | 看是否需要用户维度顺序 |

选择 key 时问自己：

```text
我希望哪个业务实体维度保持顺序？
哪个字段分布足够均匀？
哪个字段能帮助排查问题？
```

---

## 七、热点 Key 问题

如果大量消息使用同一个 key，它们会进入同一个 partition。

例如所有消息都用：

```text
key = "order"
```

后果：

- 一个 partition 特别忙。
- 其他 partition 很闲。
- consumer lag 集中在一个 partition。
- 增加 consumer 也没用，因为热点 partition 只能被一个 consumer 消费。

所以 key 既要满足顺序需求，也要尽量分布均匀。

---

## 八、Partition 数量如何选择

初学阶段：

```text
3 到 6 个 partition
```

小型业务：

```text
6 到 12 个 partition
```

高吞吐日志：

```text
需要结合压测、broker 数量、消息大小、消费能力评估
```

考虑因素：

- 峰值写入 QPS。
- consumer 并行度。
- 单条消息大小。
- 是否需要顺序。
- broker 数量。
- 未来增长。

不要无脑创建几百上千个 partition。partition 过多会增加：

- broker 文件句柄。
- controller 元数据压力。
- consumer group rebalance 成本。
- 运维复杂度。

---

## 九、增加 Partition 的风险

Kafka 支持增加 topic partition 数量。

但增加后要注意：

```text
key 到 partition 的映射可能变化
```

例如原来：

```text
hash(order_id) % 3
```

增加到 6 个 partition 后：

```text
hash(order_id) % 6
```

同一个 key 之后可能进入新的 partition。

这对依赖顺序的业务可能有影响。

所以生产环境增加 partition 前要评估：

- 是否依赖 key 顺序。
- 是否有正在处理的历史消息。
- 是否需要停机或灰度。
- consumer 是否能处理乱序或重复。

---

## 十、命令实验：观察 Partition

创建 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --if-not-exists \
  --topic stage2.partition.demo \
  --partitions 3 \
  --replication-factor 1
```

查看 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic stage2.partition.demo
```

发送带 key 的消息：

```bash
kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic stage2.partition.demo \
  --property parse.key=true \
  --property key.separator=:
```

输入：

```text
order-1:created
order-1:paid
order-1:shipped
order-2:created
order-2:paid
order-3:created
```

消费并打印 partition：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic stage2.partition.demo \
  --from-beginning \
  --property print.key=true \
  --property print.partition=true \
  --property print.offset=true
```

观察：

- 同一个 key 的消息通常在同一个 partition。
- offset 在每个 partition 内单独递增。

---

## 十一、Go 后端工程视角

在 Go producer 中，发送订单事件时应该明确设置 key：

```go
record := &kgo.Record{
    Topic: "order.created",
    Key:   []byte(orderID),
    Value: payload,
}
```

不要偷懒不设置 key。

如果不设置 key，客户端可能轮询或随机分配 partition。这样同一订单的多条事件可能进入不同 partition。

日志中也要记录 key：

```text
topic=order.created key=order_1001 event_id=evt_1001
```

当线上某个 partition lag 很高时，你可以根据 key 分析是否有热点业务对象。

---

## 十二、本节练习

1. 创建一个 3 partition 的 topic。
2. 使用 `order_id` 作为 key 发送 10 条订单消息。
3. 打印每条消息的 partition 和 offset。
4. 观察相同 key 是否进入同一个 partition。
5. 思考 `payment.succeeded` 应该用 `order_id` 还是 `payment_id` 做 key。
6. 思考如果所有消息都使用 `key=order` 会发生什么。
7. 用自己的话解释：为什么 partition 数量是 consumer group 并行度的上限。

---

## 十三、本节小结

- topic 是逻辑分类，partition 是物理分片。
- partition 提升 Kafka 的写入、存储和消费并行能力。
- Kafka 只保证 partition 内有序，不保证多 partition 全局有序。
- message key 通常决定消息进入哪个 partition。
- Go 后端中应根据业务顺序需求选择 key。
- 热点 key 会导致单个 partition lag 特别高。
- partition 数量不是越多越好。
- 增加 partition 可能改变 key 到 partition 的映射，需要谨慎。

