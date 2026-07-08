# 5. Segment、Retention 与 Log Compaction

本节目标：理解 Kafka 消息如何持久化、什么时候会被删除、retention 和 log compaction 分别适合什么场景。

很多人把 Kafka 当作“消息被消费后就删除”的队列来理解，这是不准确的。Kafka 更像追加写日志。消息是否被删除，主要由 topic 的保留策略决定，而不是某个 consumer 是否已经读过。

---

## 一、Kafka 是追加写日志

Kafka 的 partition 底层可以理解成一段不断追加的日志。

```text
partition-0 log:
  offset 0
  offset 1
  offset 2
  offset 3
```

producer 写入消息时，Kafka 把消息追加到 partition 日志末尾。

这种顺序写对磁盘非常友好，是 Kafka 高吞吐的重要原因之一。

---

## 二、Segment 是什么

Kafka 不会把一个 partition 的所有消息都写进一个无限大的文件。

它会把日志切成多个 segment。

可以理解成：

```text
partition-0:
  segment-00000000000000000000.log
  segment-00000000000000100000.log
  segment-00000000000000200000.log
```

segment 的好处：

- 便于删除过期数据。
- 便于管理大文件。
- 便于根据 offset 查找消息。
- 便于做日志清理。

初学阶段不需要深入文件格式，但要知道：

```text
Kafka 消息是持久化在磁盘上的
```

---

## 三、消息被消费后会删除吗

不会因为被消费就立刻删除。

Kafka 的删除主要由 topic 配置决定：

- 时间保留。
- 大小保留。
- compact 策略。

这就是为什么多个 consumer group 可以独立读取同一份消息。

如果消息被第一个 consumer 读完就删除，其他 group 就无法再读了。

---

## 四、Retention 是什么

Retention 表示消息保留策略。

常见配置：

```text
retention.ms
retention.bytes
```

`retention.ms` 控制消息保留多久。

`retention.bytes` 控制日志最多保留多大。

例如：

```bash
kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --alter \
  --entity-type topics \
  --entity-name stage2.retention.demo \
  --add-config retention.ms=60000
```

表示保留约 60 秒。

注意：

```text
Kafka 清理不是毫秒级立刻发生
```

它还受 segment、清理周期等影响。

---

## 五、Retention 的业务意义

如果 topic retention 是 1 天，而某个 consumer group 停止了 3 天，那么它恢复后可能读不到停机期间最早的消息。

这意味着：

```text
retention 必须覆盖业务可接受的最长恢复时间
```

例如：

- 订单事件：可能保留 7 天、30 天甚至更久。
- 用户行为日志：可能保留 3 天或 7 天，然后进入数仓。
- 临时任务事件：可能保留 1 天。

保留时间越长，磁盘成本越高。

---

## 六、Log Compaction 是什么

Log compaction 是另一种清理策略。

它不是按时间删除旧消息，而是按 key 保留较新的值。

例如，一个用户状态 topic：

```text
key=user-1 value=name=A
key=user-1 value=name=B
key=user-1 value=name=C
```

compaction 后，旧值可能被清理，最终保留较新的：

```text
key=user-1 value=name=C
```

注意是“可能”和“最终”，不是写入后立即只剩最后一条。

---

## 七、Retention 和 Compaction 的区别

| 策略 | 清理依据 | 适合场景 |
| --- | --- | --- |
| retention | 时间或大小 | 事件历史、日志、订单流水 |
| compaction | key 的最新值 | 状态同步、配置、用户最新状态 |

订单事件通常不应该只保留最新状态。

例如：

```text
order.created
order.paid
order.shipped
order.finished
```

这些都是有意义的历史事件。

如果只保留某个 key 的最新值，审计和状态回放可能出问题。

---

## 八、Tombstone 消息

在 compacted topic 中，如果发送：

```text
key=user-1 value=null
```

这类消息通常叫 tombstone。

它表示这个 key 被删除。

Kafka compaction 后可以清理这个 key 的旧值。

初学阶段只需要知道：

```text
compaction 是按 key 管理最新状态的一种机制
```

不要把它和普通事件 topic 混用。

---

## 九、命令实验：Retention

创建短保留 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --if-not-exists \
  --topic stage2.retention.demo \
  --partitions 1 \
  --replication-factor 1 \
  --config retention.ms=60000
```

发送消息：

```bash
echo "message-will-expire" | kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic stage2.retention.demo
```

立即消费：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic stage2.retention.demo \
  --from-beginning \
  --timeout-ms 10000
```

等待一段时间后再次消费。

注意：

- 不一定 60 秒后立刻消失。
- Kafka 按 segment 和清理周期处理。
- 本地实验中要耐心观察。

---

## 十、查看 Topic 配置

```bash
kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --entity-type topics \
  --entity-name stage2.retention.demo
```

你可以看到 topic 级别配置。

---

## 十一、Go 后端工程视角

设计 topic 时要问：

```text
这个事件需要保留多久？
consumer 最长可能停机多久？
是否需要历史重放？
是否有审计要求？
磁盘成本能接受吗？
```

对于 `order.created`：

```text
通常使用 retention，不使用 compaction。
```

对于 `user.latest_profile`：

```text
可以考虑 compaction。
```

对于日志类 topic：

```text
可以短期保留，之后进入 ClickHouse、对象存储或数仓。
```

---

## 十二、常见误区

### 1. 消费后消息就删除

不是。

Kafka 按保留策略删除。

### 2. retention.ms 到了就精确删除

不是。

清理受 segment 和后台清理线程影响。

### 3. compaction 适合所有 topic

不是。

compaction 适合状态，不适合完整事件历史。

### 4. retention 越长越好

不是。

retention 越长，磁盘成本越高，恢复和重放也更需要治理。

---

## 十三、本节练习

1. 创建一个 retention 为 1 分钟的 topic。
2. 写入 3 条消息。
3. 立即消费确认消息存在。
4. 等待一段时间后再次消费。
5. 查看 topic 配置。
6. 为 `order.created` 设计 retention。
7. 为用户最新状态 topic 判断是否适合 compaction。
8. 用自己的话解释 retention 和 compaction 的区别。

---

## 十四、本节小结

- Kafka partition 底层是追加写日志。
- partition 日志会切成多个 segment。
- 消息不会因为被消费就立即删除。
- retention 按时间或大小保留消息。
- retention 必须覆盖业务恢复窗口。
- log compaction 按 key 保留较新的值。
- 事件历史类 topic 通常使用 retention。
- 状态同步类 topic 可以考虑 compaction。
- Go 后端设计 topic 时必须同时考虑业务恢复和磁盘成本。

---

## 十五、Retention 实验命令示例

创建短保留 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic retention.demo \
  --partitions 1 \
  --replication-factor 1 \
  --config retention.ms=60000
```

查看配置：

```bash
kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name retention.demo \
  --describe
```

发送消息后等待超过 1 分钟，再尝试从头消费。你会发现旧消息可能已经被清理。

---

## 十六、Go 后端设计提醒

订单事件不要随便设置很短 retention。因为库存服务、通知服务、数据同步服务都可能需要停机恢复。

一个常见思路：

```text
关键业务事件：保留 7 天或更久。
日志分析事件：按成本保留 1 到 3 天。
状态快照 topic：考虑 compaction。
```

retention 是业务恢复窗口，不只是 Kafka 磁盘参数。

---

## 十七、设计对比表

| Topic | 清理策略 | 原因 |
| --- | --- | --- |
| `order.created` | retention | 需要保留事件历史 |
| `payment.succeeded` | retention | 需要审计和重放 |
| `user.latest_profile` | compaction | 只关心每个 user 最新状态 |
| `sku.latest_stock` | compaction | 适合状态同步 |

选择 retention 还是 compaction，要看业务需要事件历史还是最新状态。

---

## 十八、最终验收

完成本节后，应能说明：

```text
为什么 consumer 读过消息后，Kafka 仍然可能保留这条消息？
```

---

## 十九、最终练习

为两个 topic 设计清理策略：

```text
order.created
user.latest_profile
```

分别说明为什么选择 retention 或 compaction。

---

## 二十一、选择 retention 还是 compaction

可以用下面的判断题训练自己：

| 场景 | 更适合 | 原因 |
| --- | --- | --- |
| 订单创建事件流水 | retention | 每条历史事件都有审计价值 |
| 用户当前资料快照 | compaction | 只关心同一个 key 的最新值 |
| 支付回调记录 | retention | 历史回调需要追踪 |
| 商品当前库存快照 | compaction | 下游通常需要最新状态 |

注意：compaction 不是删除“旧文件”，而是在同一个 key 的多条记录中尽量保留最新值。

如果业务需要完整审计链路，不要为了省空间轻易改成 compacted topic。
