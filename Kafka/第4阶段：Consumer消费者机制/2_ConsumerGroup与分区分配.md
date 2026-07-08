# 2. Consumer Group 与分区分配

本节目标：理解 consumer group 如何让多个 consumer 协作消费同一个 topic，以及 partition 是如何分配给 consumer 的。

Consumer group 是 Kafka 最重要的概念之一。它既能让多个服务实例并行消费，也能让不同业务服务独立消费同一份事件流。

---

## 一、Consumer Group 是什么

Consumer group 是一组 consumer 的逻辑名称。

例如：

```text
group.id = inventory-service
```

表示库存服务这个消费者组。

如果你部署了 3 个库存服务实例：

```text
inventory-service instance-1
inventory-service instance-2
inventory-service instance-3
```

它们使用同一个 group id，就会协作消费。

---

## 二、同一个 Group 内是分工消费

假设 topic 有 3 个 partition：

```text
order.created
  partition-0
  partition-1
  partition-2
```

group 中有 3 个 consumer：

```text
consumer-A -> partition-0
consumer-B -> partition-1
consumer-C -> partition-2
```

每个 partition 同一时刻只会分配给 group 内一个 consumer。

---

## 三、Consumer 数量少于 Partition

topic 有 3 个 partition，group 有 2 个 consumer：

```text
consumer-A -> partition-0, partition-1
consumer-B -> partition-2
```

一个 consumer 可以消费多个 partition。

---

## 四、Consumer 数量多于 Partition

topic 有 3 个 partition，group 有 4 个 consumer：

```text
consumer-A -> partition-0
consumer-B -> partition-1
consumer-C -> partition-2
consumer-D -> 空闲
```

这就是为什么：

```text
同一个 group 的最大并行度 <= partition 数量
```

增加 consumer 不一定提升吞吐。

---

## 五、不同 Group 之间独立消费

同一个 topic 可以被多个 group 消费：

```text
order.created
  -> inventory-service
  -> notification-service
  -> analytics-service
```

这些 group 各自有自己的 offset。

库存服务消费完，不影响通知服务。

这就是 Kafka 事件流模型和普通队列的一个重要区别。

---

## 六、命令实验：多个 Consumer

创建 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --if-not-exists \
  --topic stage4.group.demo \
  --partitions 3 \
  --replication-factor 1
```

启动第一个 consumer：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic stage4.group.demo \
  --group stage4-group \
  --property print.partition=true \
  --property print.offset=true
```

再启动第二个、第三个相同命令的 consumer。

发送消息：

```bash
for i in $(seq 1 20); do
  echo "order-$i:created-$i"
done | kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic stage4.group.demo \
  --property parse.key=true \
  --property key.separator=:
```

观察：

- 多个 consumer 会分摊 partition。
- 同一个 partition 不会同时被两个 consumer 处理。

---

## 七、查看 Group 状态

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group stage4-group
```

观察字段：

- `TOPIC`
- `PARTITION`
- `CURRENT-OFFSET`
- `LOG-END-OFFSET`
- `LAG`
- `CONSUMER-ID`
- `HOST`
- `CLIENT-ID`

如果某个 partition 没有 consumer，说明当前 group 可能没有运行实例，或正在 rebalance。

---

## 八、Go 后端部署视角

假设 `order.created` 有 6 个 partition。

你部署：

```text
inventory-service replicas=3
```

每个实例平均消费 2 个 partition。

如果扩容到：

```text
inventory-service replicas=6
```

每个实例平均消费 1 个 partition。

如果扩容到：

```text
inventory-service replicas=10
```

只有 6 个实例真正消费，4 个空闲。

所以扩容前要先看 partition 数。

---

## 九、Group ID 命名建议

推荐使用服务名：

```text
inventory-service
notification-service
analytics-service
```

不要使用随机 group id，除非你明确想每次从头消费。

如果每次启动都生成新 group id：

```text
inventory-service-随机数
```

结果：

- Kafka 认为这是新 group。
- 可能从 earliest 或 latest 重新开始。
- offset 无法延续。

---

## 十、常见误区

### 1. 多个 group 会抢同一条消息

不会。

不同 group 独立消费。

### 2. 同一个 group 中 consumer 越多越好

不一定。

超过 partition 数就会空闲。

### 3. Group ID 随便填

不应该。

group id 决定消费进度归属。

### 4. 一个 partition 可以同时被同 group 多个 consumer 消费

不能。

同一时刻只能分配给一个。

---

## 十一、本节练习

1. 创建 3 partition topic。
2. 启动 1 个 consumer，观察它消费几个 partition。
3. 启动 2 个 consumer，观察分配变化。
4. 启动 4 个 consumer，观察是否有空闲。
5. 换一个 group id，从头消费同一个 topic。
6. 解释为什么不同服务应该使用不同 group id。

---

## 十二、本节小结

- Consumer group 是 Kafka 协作消费的基本单位。
- 同一个 group 内是分工消费。
- 不同 group 之间独立消费。
- 一个 partition 同一时刻只能分配给同 group 内一个 consumer。
- consumer 并行度受 partition 数限制。
- Go 服务扩容前要先看 topic partition 数。
- group id 应该稳定，通常使用服务名。

---

## 十四、分区分配实验

创建 3 分区 topic，然后启动 1 个 consumer：

```text
consumer-1: partition 0,1,2
```

启动 2 个 consumer，可能变成：

```text
consumer-1: partition 0,1
consumer-2: partition 2
```

启动 4 个 consumer：

```text
consumer-1: partition 0
consumer-2: partition 1
consumer-3: partition 2
consumer-4: idle
```

这说明 consumer 数超过 partition 数后，多出来的实例不会增加吞吐。

---

## 十五、Go 服务扩容判断

扩容前先问：

```text
当前 topic 有多少 partition？
每个 partition lag 是否均匀？
handler 是 CPU 慢、DB 慢，还是外部接口慢？
```

如果只有一个热点 partition，盲目加 consumer 通常没有效果。

---

## 十六、面试表达

```text
Consumer group 内部是分工消费，同一个 partition 在同一时刻只会分配给组内一个 consumer。
所以扩容 consumer 的上限受 partition 数限制。
如果 lag 集中在单个 partition，需要先排查 key 分布，而不是只加实例。
```

---

## 十七、实验记录模板

建议做一次分配实验并记录：

```text
topic: group.assignment.demo
partitions: 3
group: inventory-service

1 consumer:
  consumer-a -> partition 0,1,2

2 consumers:
  consumer-a -> partition 0,1
  consumer-b -> partition 2

4 consumers:
  consumer-a -> partition 0
  consumer-b -> partition 1
  consumer-c -> partition 2
  consumer-d -> idle
```

具体分配结果可能因策略不同而不同，但核心规律不变：

```text
同一个 group 内，一个 partition 同一时刻只能给一个 consumer。
```

---

## 十八、设计提醒

如果你预计库存服务需要 20 个实例并行消费，那么 topic 至少要有接近 20 个 partition。

但 partition 也不是越多越好。partition 过多会增加 broker 元数据、文件句柄、leader 选举和 rebalance 成本。

所以生产设计要结合：

- 峰值吞吐。
- 单 consumer 处理能力。
- 未来增长。
- 顺序性要求。
- 运维成本。

---

## 十九、分区扩容提醒

Kafka topic 可以增加 partition，但增加后 key 到 partition 的映射可能变化。

这意味着：

```text
新消息的分区分布可能改变。
旧消息不会重新分布。
依赖 key 顺序的业务要谨慎评估。
```

所以初始 partition 规划要留余量，不要完全按当前最小流量设计。

---

## 二十、最终验收

完成本节后，应能解释：

```text
为什么 3 partition 的 topic，启动 10 个同组 consumer 也只有 3 个会真正消费？
```

---

## 二十、扩容前先看 partition

Consumer 扩容不是无限加实例。

```text
topic partition 数 = 3
consumer 实例数 = 1 -> 最多 1 个实例消费
consumer 实例数 = 3 -> 最多 3 个实例并行消费
consumer 实例数 = 10 -> 仍然最多 3 个实例真正消费
```

因此，当你发现 lag 很高时，不能直接说“多加几台 consumer”。

正确顺序是：先看 partition 数，再看每个 partition 的 lag，再看单个 handler 的处理耗时，最后决定是优化处理逻辑、增加 partition，还是增加 consumer 实例。

---

## 二十一、分配结果记录

每次扩容 consumer，都记录一次分配结果：

```text
topic：
partition 数：
group：
consumer 实例数：
每个实例分到的 partition：
扩容前总 lag：
扩容后总 lag：
是否发生长时间 rebalance：
```

这份记录能帮助你判断扩容是否真的提高了处理能力，而不是只增加了空闲实例。
