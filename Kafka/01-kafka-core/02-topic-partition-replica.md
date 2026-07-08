# 02 Topic、Partition 与 Replica

本节目标：理解 Kafka 如何通过 partition 和 replica 同时获得并行能力和容错能力。

## Topic 是逻辑分类

topic 是业务事件的分类。

推荐命名：

- `order.created`
- `order.cancelled`
- `payment.succeeded`
- `inventory.deducted`

不推荐：

- `order_queue`
- `test1`
- `send_message`
- `handle_order`

原因是 Kafka 中的消息最好表示“已经发生的事实”，而不是“命令某个服务去做什么”。

## Partition 是物理分片

一个 topic 可以拆成多个 partition。

例如：

```text
topic: order.created
  partition-0
  partition-1
  partition-2
```

消息写入哪个 partition，通常由 key 决定。

如果 key 是 `order_id`，那么同一个订单的事件通常会进入同一个 partition，从而保证同一订单维度的顺序。

## Partition 决定并行度

在一个 consumer group 内：

- 1 个 partition 同一时刻只能被 1 个 consumer 消费。
- 1 个 consumer 可以消费多个 partition。
- consumer 数量大于 partition 数量时，多出来的 consumer 会空闲。

例子：

```text
topic 有 3 个 partition
consumer group 有 2 个 consumer

consumer-A: partition-0, partition-1
consumer-B: partition-2
```

如果 group 有 4 个 consumer：

```text
consumer-A: partition-0
consumer-B: partition-1
consumer-C: partition-2
consumer-D: 空闲
```

所以，不是 consumer 越多吞吐越高。上限首先取决于 partition 数量。

## Replica 是副本

replica 用来提高可用性。

如果一个 topic 的 replication factor 是 3，那么每个 partition 会有 3 份副本，分布在不同 broker 上。

```text
partition-0:
  replica on broker-1
  replica on broker-2
  replica on broker-3
```

其中一个 replica 是 leader，其他是 follower。

## Leader 与 Follower

producer 和 consumer 通常与 partition leader 交互。

follower 从 leader 复制数据。

如果 leader 所在 broker 宕机，Kafka 会从可用 follower 中选出新的 leader。

## ISR 是什么

ISR 是 in-sync replicas，表示与 leader 保持同步的副本集合。

如果 follower 落后太多，可能会被移出 ISR。

生产中需要关注：

- ISR 数量是否下降。
- 是否有 under replicated partitions。
- 是否有 offline partitions。

## 创建带多个 partition 的 topic

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic order.created \
  --partitions 3 \
  --replication-factor 1
```

本地单节点只能用 `--replication-factor 1`。多 broker 环境才适合设置为 3。

## 什么时候增加 partition

适合增加 partition：

- producer 写入吞吐不够。
- consumer 并行度不够。
- 单个 partition 数据量过大。

需要谨慎：

- 增加 partition 后，同一个 key 的分区映射可能变化。
- 顺序性可能受到影响。
- partition 过多会增加 broker、controller、consumer group 的管理开销。

## 本节练习

1. 创建一个 3 partition 的 topic。
2. 启动 1 个 consumer，观察它消费几个 partition。
3. 启动 3 个 consumer，观察分配结果。
4. 启动 4 个 consumer，确认是否有 consumer 空闲。
5. 用自己的话解释：为什么 partition 数量是消费并行度的上限？

