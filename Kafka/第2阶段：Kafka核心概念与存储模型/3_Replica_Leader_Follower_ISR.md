# 3. Replica、Leader、Follower 与 ISR

本节目标：理解 Kafka 如何通过副本提升可用性，掌握 replica、leader、follower、ISR、under replicated partitions 这些生产环境必须看懂的概念。

第 1 阶段我们使用的是单节点 Kafka，所以每个 partition 只有一个副本。真实生产环境一般不会这样，因为单节点挂掉后整个 Kafka 就不可用。Kafka 的高可用依赖多个 broker 和副本复制。

---

## 一、为什么需要副本

假设一个 topic 只有一个 partition，并且只有一份数据：

```text
broker-1
  order.created partition-0
```

如果 `broker-1` 宕机：

- producer 无法继续写这个 partition。
- consumer 无法读取这个 partition。
- 如果磁盘损坏，数据可能丢失。

所以生产环境需要副本：

```text
partition-0 replica on broker-1
partition-0 replica on broker-2
partition-0 replica on broker-3
```

这就是 replication factor。

---

## 二、Replica 是什么

Replica 是 partition 的副本。

如果一个 topic 有：

```text
partitions = 3
replication factor = 3
```

那么总副本数是：

```text
3 partitions * 3 replicas = 9 replicas
```

每个 partition 有 3 份副本，通常分布在不同 broker 上。

---

## 三、Leader 和 Follower

每个 partition 的多个 replica 中，有一个是 leader，其他是 follower。

```text
partition-0:
  broker-1 leader
  broker-2 follower
  broker-3 follower
```

producer 和 consumer 通常与 leader 通信。

follower 从 leader 复制数据。

如果 leader 所在 broker 挂了，Kafka 会从合适的 follower 中选出新的 leader。

---

## 四、为什么不让所有副本都处理写入

如果所有副本都同时接收写入，就要解决多副本写入顺序和冲突问题，复杂度会很高。

Kafka 的做法是：

```text
一个 partition 只有一个 leader 负责读写。
follower 只复制 leader 的数据。
```

这样每个 partition 内的日志顺序由 leader 统一决定。

---

## 五、ISR 是什么

ISR 是 in-sync replicas。

可以理解为：

```text
当前和 leader 保持同步的一组副本
```

例如：

```text
partition-0:
  leader: broker-1
  replicas: broker-1, broker-2, broker-3
  isr: broker-1, broker-2, broker-3
```

如果 broker-3 落后太多，可能被移出 ISR：

```text
isr: broker-1, broker-2
```

ISR 变少说明副本同步不健康。

---

## 六、acks 与 ISR 的关系

producer 的 `acks=all` 表示：

```text
leader 等待 ISR 中满足要求的副本确认后，再向 producer 返回成功
```

生产环境常见组合：

```text
replication.factor=3
min.insync.replicas=2
producer acks=all
```

含义：

- 每个 partition 有 3 个副本。
- 至少 2 个同步副本可用时才允许确认写入。
- producer 等待足够副本确认。

这样即使一个 broker 挂了，仍然可以保持较好的可靠性。

---

## 七、Under Replicated Partitions

Under replicated partition 表示：

```text
某个 partition 的 ISR 数量少于 replica 数量
```

例如：

```text
replicas: broker-1, broker-2, broker-3
isr: broker-1, broker-2
```

说明 broker-3 的副本没有跟上。

生产环境中这个指标非常重要。

可能原因：

- broker 宕机。
- broker 网络慢。
- broker 磁盘慢。
- follower 复制跟不上。
- broker GC 或 CPU 压力大。

---

## 八、Offline Partitions

Offline partition 更严重。

它表示某个 partition 没有可用 leader。

后果：

- producer 不能写。
- consumer 不能读。
- 业务链路中断。

如果线上出现 offline partitions，需要立即处理。

---

## 九、本地单节点环境能看到什么

在本地单节点环境中，我们只能设置：

```text
replication-factor = 1
```

查看 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic quickstart.orders
```

你会看到：

```text
Leader: 1
Replicas: 1
Isr: 1
```

这表示：

- leader 是 broker 1。
- 只有一个 replica。
- ISR 中也只有 broker 1。

这没有高可用能力，但足够学习其他概念。

---

## 十、为什么生产环境不推荐副本因子为 1

如果 replication factor 是 1：

- broker 挂了，partition 不可用。
- 磁盘坏了，数据可能丢失。
- 无法容忍单点故障。

关键业务 topic 通常使用：

```text
replication.factor=3
```

比如：

- 订单事件。
- 支付事件。
- 库存事件。
- 审计事件。

---

## 十一、Go 后端工程视角

Go producer 不能只靠自己保证消息可靠。

可靠性需要多层配合：

```text
topic replication factor
broker ISR 健康
producer acks
producer retry
consumer 幂等
监控告警
```

如果 topic 只有一个副本，即使 producer 设置 `acks=all`，也只是等待一个副本确认。

所以面试或生产设计时，不要只说：

```text
我设置 acks=all，所以消息不会丢
```

更完整的说法是：

```text
关键 topic 使用 replication.factor=3，min.insync.replicas=2，producer 使用 acks=all，并监控 under replicated partitions。
```

---

## 十二、常见误区

### 1. replication factor 越高越好

不是。

副本越多，可靠性更高，但存储成本、网络复制成本也更高。

常见生产设置是 3，不是无限增加。

### 2. follower 也负责同等写入

不是。

partition 的写入由 leader 处理，follower 复制 leader。

### 3. ISR 等于所有 replica

不一定。

落后的 follower 会被移出 ISR。

### 4. acks=all 就一定不丢消息

不绝对。

它还依赖 ISR、min.insync.replicas、unclean leader election、磁盘可靠性等配置和实际状态。

---

## 十三、本节练习

1. 查看 `quickstart.orders` 的 leader、replicas、ISR。
2. 解释为什么本地单节点 `replication-factor` 只能是 1。
3. 假设生产环境有 3 个 broker，为 `order.created` 设计副本因子。
4. 解释 `replication.factor=3` 和 `min.insync.replicas=2` 的意义。
5. 思考：如果 ISR 从 3 变成 1，producer 还应该继续写入关键消息吗？
6. 用自己的话解释 leader 和 follower 的区别。

---

## 十四、本节小结

- replica 是 partition 的副本。
- 每个 partition 有一个 leader，其他副本是 follower。
- producer 和 consumer 通常与 leader 通信。
- follower 从 leader 复制数据。
- ISR 表示当前同步中的副本集合。
- under replicated partitions 表示副本同步不健康。
- offline partitions 表示没有可用 leader，问题更严重。
- 关键 topic 生产环境通常使用 replication factor 3。
- Go producer 的可靠性必须和 topic 副本配置一起看。

---

## 十四、排查命令补充

查看 topic 副本状态：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic order.created
```

重点看：

```text
Leader
Replicas
Isr
```

如果 `Replicas` 有 3 个 broker，但 `Isr` 只有 1 个，说明副本同步不健康。

---

## 十五、Go 后端视角

Go producer 设置 `acks=all` 只是客户端侧要求。它还依赖 broker 侧：

```text
replication.factor
min.insync.replicas
ISR 健康状态
```

如果 topic 只有一个副本，即使 producer 等到了 ack，也不代表具备高可用。

---

## 十六、生产排查命令

查看所有 topic 是否有副本不足：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --under-replicated-partitions
```

查看是否有不可用 partition：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --unavailable-partitions
```

学习环境可能没有这些问题，但生产环境这两个命令非常重要。

---

## 十七、面试表达

```text
Kafka 的 partition 可以有多个 replica，其中 leader 负责读写，follower 从 leader 复制。
ISR 表示当前和 leader 保持同步的副本集合。
如果 ISR 变少，说明副本同步不健康；如果 partition 没有 leader，就无法正常读写。
```

---

## 十八、生产告警建议

生产环境至少关注：

```text
UnderReplicatedPartitions
OfflinePartitionsCount
ActiveControllerCount
ISR shrink 次数
```

如果 offline partition 出现，优先级要高于普通 lag 告警，因为它表示分区可能无法读写。

---

## 二十一、用故障故事理解 ISR

可以用下面的小故事复盘 ISR：

```text
一个 topic 有 3 个副本。
leader 正常接收写入。
一个 follower 因为网络抖动追不上 leader。
它被移出 ISR。
如果 acks=all，写入只等待仍在 ISR 中的副本确认。
如果 ISR 数量低于 min.insync.replicas，写入可能失败。
```

这段故事说明：ISR 不是“所有副本列表”，而是“当前仍能跟上 leader 的同步副本集合”。

做 Go 后端时，看到生产者返回不可用副本相关错误，不要只重试，要同时查看集群 ISR 是否持续收缩。
