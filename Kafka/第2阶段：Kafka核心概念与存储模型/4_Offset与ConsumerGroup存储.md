# 4. Offset 与 Consumer Group 存储

本节目标：理解 offset 的含义、consumer group 如何记录消费进度、为什么 offset 提交时机决定消息是否可能丢失或重复。

Offset 是 Kafka consumer 的核心。Go 后端工程师只要写 consumer，就必须真正理解 offset。很多线上问题，比如重复消费、消息丢失、lag 不下降，都和 offset 有关。

---

## 一、Offset 是什么

Offset 是消息在某个 partition 内的位置。

例如：

```text
topic: order.created

partition-0:
  offset 0: order-1 created
  offset 1: order-2 created
  offset 2: order-3 created

partition-1:
  offset 0: order-4 created
  offset 1: order-5 created
```

注意：

```text
offset 只在 partition 内有意义
```

`partition-0 offset=1` 和 `partition-1 offset=1` 是两条不同消息。

---

## 二、Consumer Group 为什么需要 Offset

consumer group 是一组协作消费同一个 topic 的 consumer。

Kafka 需要知道：

```text
这个 group 在每个 partition 上消费到了哪里
```

例如：

```text
group: inventory-service
topic: order.created
partition: 0
committed offset: 101
```

这表示：

```text
inventory-service 下次应该从 partition 0 的某个位置继续消费
```

不同 group 有不同 offset。

---

## 三、为什么 Offset 属于 Group

同一个 topic 可能被多个服务消费：

```text
order.created
  -> inventory-service
  -> notification-service
  -> analytics-service
```

库存服务消费完，不应该影响通知服务和分析服务。

所以 Kafka 的消费进度不是 topic 级别的，而是：

```text
group + topic + partition
```

这也是 Kafka 支持事件广播的重要基础。

---

## 四、Committed Offset 的含义

提交 offset 的本质是告诉 Kafka：

```text
这个 consumer group 已经处理到这里了
```

在 Go 后端里，推荐流程是：

```text
拉取消息
解析消息
执行业务逻辑
业务成功
提交 offset
```

如果业务没成功就提交 offset，可能造成消息处理丢失。

---

## 五、提交的是当前 Offset 还是下一个 Offset

Kafka 中 committed offset 通常表示：

```text
下一条要消费的位置
```

例如你处理了 offset 10 的消息，提交的通常是 11。

不同客户端 API 会帮你处理这个细节。

作为应用开发者，更重要的是理解语义：

```text
只有当 offset 10 对应的消息业务处理成功后，才能让 group 跳过它继续往后
```

---

## 六、自动提交与手动提交

### 自动提交

客户端定期自动提交 offset。

优点：

- 使用简单。
- 适合学习或非关键日志场景。

缺点：

- 可能业务还没处理成功，offset 已经提交。
- 失败处理不精细。
- 不适合关键业务。

### 手动提交

应用代码在业务成功后提交 offset。

优点：

- 可靠性更可控。
- 能配合 retry、DLQ、幂等。

缺点：

- 代码复杂。
- 必须处理 commit 失败、重复消费、优雅退出。

生产 Go consumer 通常使用手动提交。

---

## 七、Offset 存在哪里

Kafka 会把 consumer group 的 offset 存到内部 topic：

```text
__consumer_offsets
```

你平时不需要直接读写这个 topic。

但要知道：

```text
consumer group offset 是 Kafka 自己管理的一类消息数据
```

这也是为什么本地单节点配置里有：

```yaml
KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```

它控制 `__consumer_offsets` 这个内部 topic 的副本因子。

生产环境中这个内部 topic 也需要可靠存储。

---

## 八、Consumer Lag

Lag 表示 consumer group 落后多少消息。

简化公式：

```text
lag = log end offset - committed offset
```

例如：

```text
LOG-END-OFFSET = 1000
CURRENT-OFFSET = 900
LAG = 100
```

说明这个 group 在这个 partition 上还落后 100 条消息。

Lag 高不一定就是 Kafka 问题，它可能说明：

- producer 写入突然增加。
- consumer 处理变慢。
- consumer 报错重试。
- 下游数据库慢。
- 某个 partition 有热点 key。
- consumer group 频繁 rebalance。

---

## 九、命令实验：观察 Offset 与 Lag

创建 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --if-not-exists \
  --topic stage2.offset.demo \
  --partitions 3 \
  --replication-factor 1
```

发送消息：

```bash
for i in $(seq 1 10); do
  echo "order-$i:created-$i"
done | kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic stage2.offset.demo \
  --property parse.key=true \
  --property key.separator=:
```

使用 group 消费：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic stage2.offset.demo \
  --group stage2-offset-group \
  --from-beginning
```

消费完后停止 consumer。

查看 group：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group stage2-offset-group
```

继续发送消息但不启动 consumer：

```bash
for i in $(seq 11 20); do
  echo "order-$i:created-$i"
done | kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic stage2.offset.demo \
  --property parse.key=true \
  --property key.separator=:
```

再次查看 group，观察 lag 增长。

---

## 十、Go 后端工程视角

一个可靠 consumer 的主流程应该是：

```text
poll message
handle business
if success:
    commit offset
if retryable error:
    send retry topic
    if retry message sent:
        commit offset
if non-retryable error:
    send DLQ
    if DLQ message sent:
        commit offset
```

不能写成：

```text
poll message
commit offset
handle business
```

这种写法业务失败时会丢处理机会。

---

## 十一、提交 Offset 失败怎么办

如果业务处理成功，但提交 offset 失败，会发生什么？

重启后可能再次消费这条消息。

所以：

```text
consumer handler 必须幂等
```

例如库存扣减：

- 使用 `event_id` 写入 `processed_events` 表。
- 如果事件已处理，直接返回成功。
- 然后再次提交 offset。

---

## 十二、常见误区

### 1. 只要 consumer 打印了消息，就表示处理成功

不是。

打印消息不等于业务成功，也不等于 offset 提交成功。

### 2. Lag 高一定是 Kafka 慢

不是。

大多数 lag 问题来自 consumer 业务处理慢或下游依赖慢。

### 3. 自动提交适合所有场景

不是。

订单、支付、库存等关键业务应该使用手动提交。

### 4. 提交 offset 后还能自动回到之前位置

不能依赖这种行为。

如果错误提交了 offset，通常需要人工 reset offset 或通过 DLQ/补偿处理。

---

## 十三、本节练习

1. 创建一个 topic 并发送 20 条消息。
2. 使用新 group 消费 10 条后停止。
3. 查看 `CURRENT-OFFSET`、`LOG-END-OFFSET`、`LAG`。
4. 继续发送 10 条消息。
5. 再次查看 lag。
6. 重启 consumer，观察 lag 下降。
7. 用自己的话解释为什么 offset 属于 consumer group。
8. 写出“业务成功后提交 offset”的伪代码。

---

## 十四、本节小结

- offset 是消息在 partition 内的位置。
- offset 不是 topic 全局唯一位置。
- consumer group 会记录自己在每个 partition 上的 offset。
- 不同 group 独立维护 offset。
- committed offset 表示 group 的消费进度。
- 自动提交简单但不适合关键业务。
- 手动提交更可靠，但要求业务幂等。
- lag 表示 group 落后多少消息。
- Go consumer 应在业务成功后提交 offset。

---

## 十五、Offset 排查命令

查看 group 进度：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group inventory-service
```

重点看：

```text
CURRENT-OFFSET
LOG-END-OFFSET
LAG
```

它们能告诉你 consumer 到底落后多少。

---

## 十六、最终验收

完成本节后，应能解释：

```text
为什么同一个 topic 可以被 inventory-service 和 notification-service 两个 group 独立消费？
```
