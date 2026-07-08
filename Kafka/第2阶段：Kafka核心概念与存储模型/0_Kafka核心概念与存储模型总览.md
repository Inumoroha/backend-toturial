# 第 2 阶段：Kafka 核心概念与存储模型总览

第 1 阶段已经能把消息发进去、读出来。第 2 阶段要开始理解 Kafka 内部的核心对象：broker、controller、topic、partition、replica、leader、follower、ISR、offset、segment、retention 和 log compaction。

这一阶段不是为了背名词，而是为了回答后端项目中最常见的问题：

```text
为什么 partition 数量会影响并发？
为什么 Kafka 可以高吞吐？
broker 挂了会发生什么？
消息为什么不会被消费后立刻删除？
consumer 停很久以后还能不能继续读？
```

---

## 一、本阶段课程

1. `1_Broker_Controller与KRaft.md`
2. `2_Topic与Partition深入理解.md`
3. `3_Replica_Leader_Follower_ISR.md`
4. `4_Offset与ConsumerGroup存储.md`
5. `5_Segment_Retention与LogCompaction.md`
6. `6_综合实践_观察Partition与Offset变化.md`

---

## 二、本阶段学习目标

完成后你应该能做到：

- 解释 broker 和 controller 的职责。
- 理解 KRaft 是什么，以及为什么初学阶段不需要 ZooKeeper。
- 解释 topic 和 partition 的关系。
- 解释 replica、leader、follower、ISR。
- 解释 offset 为什么只在 partition 内有意义。
- 解释 consumer group offset 存在哪里。
- 理解 Kafka 顺序写日志的存储模型。
- 区分 retention 和 log compaction。

---

## 三、本阶段 Go 后端视角

这些概念会直接影响 Go 代码：

- producer 发送消息时要选择合适 key。
- consumer group 的并行度受 partition 限制。
- 业务需要顺序时，要让同一业务 key 进入同一 partition。
- consumer 长时间停机可能遇到 retention 到期，导致消息无法再消费。
- 生产环境要监控 under replicated partitions 和 offline partitions。

---

## 四、本阶段验收

你需要能回答：

1. topic 和 partition 是什么关系？
2. 为什么一个 partition 同一时刻只能被同一个 group 中的一个 consumer 消费？
3. replica 和 partition 是什么关系？
4. ISR 变少说明什么？
5. offset 为什么不是全局递增？
6. retention 到期后，落后的 consumer 会遇到什么问题？

---

## 五、本阶段最终产出

学完本阶段，你应该能画出 Kafka 的核心结构图：

```text
Cluster
  Broker-1
  Broker-2
  Broker-3

Topic: order.created
  Partition-0
    Leader: Broker-1
    Followers: Broker-2, Broker-3
    Offset: 0,1,2,3...
  Partition-1
    Leader: Broker-2
    Followers: Broker-1, Broker-3
```

并且能解释：

- producer 写入的是 topic，但实际落到某个 partition。
- partition 内 offset 递增，不同 partition 的 offset 不能比较大小。
- consumer group 记录的是每个 partition 的消费进度。
- replica 解决的是 broker 故障时的数据可用性。
- retention 和 compaction 决定消息能保留多久、如何被清理。

这不是画图题，而是后续设计 topic、估算分区、排查 lag、处理故障的基础。

---

## 六、最小实验闭环

本阶段至少完成一次完整观察：

```text
创建 3 partition topic
-> 使用不同 key 发送消息
-> 查看消息落到哪些 partition
-> 启动 consumer group
-> 观察 offset 前进
-> 停止 consumer
-> 继续发送消息
-> 观察 lag 增长
```

常用命令：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic order.created
```

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group inventory-service
```

观察时不要只看总 lag，要看每个 partition 的 lag。

---

## 七、和 Go 后端的关系

| Kafka 概念 | Go 后端影响 |
| --- | --- |
| partition | 决定 consumer 最大并行度 |
| message key | 决定同一业务对象是否进入同一分区 |
| offset | 决定手动提交和重复消费行为 |
| replica/ISR | 决定 producer 可靠性配置是否真正有效 |
| retention | 决定 consumer 停机后还能不能补消费 |
| compaction | 适合保存按 key 更新的最新状态 |

例如订单事件如果要求同一个订单内有序，就应该使用 `order_id` 作为 key，而不是随机 key。

---

## 八、常见学习误区

### 1. 把 topic 当成数据库表

topic 更像追加日志，不是可随意查询更新的表。消息写入后主要按 offset 顺序读取。

### 2. 以为 offset 是全局递增

offset 只在 partition 内有意义。`partition 0 offset 10` 和 `partition 1 offset 10` 是两条不同消息。

### 3. 以为 consumer 消费后消息就删除

Kafka 消息是否删除由 retention 或 compaction 决定，不由某个 consumer 是否读过决定。

### 4. 只看 topic 总 lag

如果 lag 集中在一个 partition，说明可能有热点 key。总 lag 会隐藏这个问题。

---

## 九、本阶段练习建议

学完本阶段后，写一份自己的笔记，回答：

```text
1. order.created 应该设置几个 partition？为什么？
2. 如果 inventory-service 停机 2 天，retention 至少应该多长？
3. 如果 ISR 从 3 变成 1，producer 的 acks=all 还有什么风险？
4. 如果同一个 order_id 的事件落到不同 partition，会有什么问题？
```

这些问题会在第 7、8 阶段容量规划和生产设计中再次出现。

---

## 十、本阶段推荐画图

学习本阶段时，建议至少画三张图：

### 1. Topic 和 Partition 图

```text
order.created
  partition-0: offset 0,1,2
  partition-1: offset 0,1,2
  partition-2: offset 0,1,2
```

这张图帮助你理解 offset 为什么不是 topic 全局递增。

### 2. Replica 和 Leader 图

```text
partition-0
  leader: broker-1
  follower: broker-2
  follower: broker-3
```

这张图帮助你理解 producer 实际写 leader，follower 复制 leader。

### 3. Consumer Group 图

```text
inventory-service group
  consumer-a -> partition-0
  consumer-b -> partition-1
  consumer-c -> partition-2
```

这张图帮助你理解为什么 partition 数决定同组最大消费并行度。

---

## 十一、实验数据记录

建议记录一次完整实验：

| 项目 | 记录 |
| --- | --- |
| topic | `core.order.created` |
| partitions | 3 |
| message key | `order_id` |
| 发送数量 | 20 |
| consumer group | `core-inventory` |
| 最大 lag | 20 |
| 最终 lag | 0 |

记录实验数据不是形式主义。Kafka 的概念很多，只有和具体数字绑定，后面排查问题才不会空。

---

## 十二、进入第 3 阶段前

进入 producer 阶段前，你需要特别理解：

```text
producer 不是把消息写进一个抽象队列。
producer 最终会把消息写入某个 topic 的某个 partition。
```

因此第 3 阶段讲 message key、acks、batch、幂等时，都要回到第 2 阶段的 partition、leader、replica、offset 来理解。

---

## 十三、本阶段面试表达

可以这样回答 Kafka 核心模型：

```text
Kafka 的 topic 会被拆成多个 partition，每个 partition 是一段追加日志，offset 只在 partition 内递增。
producer 根据 key 或分区策略把消息写入某个 partition，consumer group 内的 consumer 分工消费这些 partition。
为了高可用，partition 可以有多个 replica，其中 leader 负责读写，follower 复制 leader，ISR 表示当前同步健康的副本集合。
```

---

## 十四、本阶段排障能力

学完本阶段后，至少应该能看懂这些现象：

```text
某个 partition lag 特别高。
consumer group 当前 offset 不再前进。
topic describe 里 ISR 数量变少。
consumer 停机太久后读不到旧消息。
compacted topic 中同一个 key 只保留较新的值。
```

这些现象后面都会出现在真实项目里。

---

## 十五、下一阶段预告

第 3 阶段会从 producer 角度继续展开：

- producer 如何选择 partition。
- message key 为什么重要。
- acks 如何影响可靠性。
- retry 为什么会带来重复。
- batch 和 compression 为什么能提升吞吐。

你会发现 producer 的很多参数，最终都要回到第 2 阶段的 partition、leader、replica 和 offset 来理解。

---

## 十六、阶段结束小作业

请用自己的话写一页文档：

```text
Kafka 为什么能让多个 consumer 并行消费？
Kafka 为什么能在 consumer 停止后继续保留消息？
Kafka 为什么需要 replica、leader、follower 和 ISR？
```

要求不要只写定义，必须结合 `order.created` 这个 topic 举例。

---

## 十七、学习产物检查

本阶段结束时，应该留下：

- 一张 topic/partition/offset 图。
- 一张 replica/leader/follower/ISR 图。
- 一张 consumer group 分配图。
- 一份 retention 和 compaction 对比笔记。
- 一次 `kafka-topics.sh --describe` 输出解读。

这些图和笔记后面可以直接复用到面试复习中。

---

## 十八、面试表达

如果面试官问 Kafka 为什么吞吐高，可以这样回答：

```text
Kafka 把 topic 拆成多个 partition，每个 partition 是追加写日志，顺序写磁盘效率高。
多个 partition 可以分布在不同 broker 上，同时也允许 consumer group 并行消费。
再加上批量发送、页缓存和零拷贝等机制，Kafka 能在高吞吐场景下保持较好的性能。
```

---

## 十九、推荐学习时间安排

| 天数 | 任务 | 重点 |
| --- | --- | --- |
| 第 1 天 | Broker、Controller、KRaft | 集群角色 |
| 第 2 天 | Topic、Partition | 并行和顺序 |
| 第 3 天 | Replica、ISR | 高可用 |
| 第 4 天 | Offset、ConsumerGroup | 消费进度 |
| 第 5 天 | Segment、Retention、Compaction | 存储清理 |

每学一个概念，都回到 `order.created` 这个例子上解释一遍。

---

## 二十、概念关系速记

```text
topic 包含多个 partition。
partition 是有序追加日志。
message 在 partition 内有 offset。
partition 可以有多个 replica。
leader replica 负责读写。
consumer group 保存每个 partition 的 offset。
retention 决定消息保留时间。
```

这段速记能完整串起第 2 阶段。

---

## 二十一、复习建议

如果后面学习 producer、consumer、lag 时卡住，就回到本阶段重新画：

```text
topic -> partition -> offset -> consumer group
```

很多 Kafka 问题，本质都是这几个对象的关系没理顺。

---

## 二十二、复盘输出

建议最终写一页：

```text
Kafka 核心对象关系说明
```

要求用 `order.created` 举例，解释 topic、partition、offset、replica、consumer group 之间的关系。

这页笔记后面可以反复复习。

---

## 二十三、检查清单

复习结束前，确认你能不看资料解释：

- `order.created` 为什么可以拆成多个 partition。
- consumer group 为什么要分别记录每个 partition 的 offset。
- replica 为什么不能替代 consumer 并行度。
- retention 为什么和 consumer 是否读过消息无关。

---

## 二十三、阶段复盘写法

本阶段学完后，建议像 PostgreSQL 表设计教程那样，把抽象概念落成一段业务说明。

可以按这个格式写：

```text
业务 topic：
预计 partition 数：
为什么这样分区：
副本数：
leader/follower 如何影响读写：
offset 属于谁：
retention 策略：
可能的故障场景：
```

如果你能用订单、库存、支付这些具体业务把每个字段填出来，说明 Kafka 存储模型已经不再只是名词。
