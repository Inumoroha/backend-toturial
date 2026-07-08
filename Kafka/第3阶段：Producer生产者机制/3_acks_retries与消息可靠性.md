# 3. acks、retries 与消息可靠性

本节目标：理解 producer 可靠发送的核心参数 `acks` 和 `retries`，知道它们分别解决什么问题、带来什么代价，以及为什么 producer 可靠性不能脱离 broker 副本配置讨论。

在 Go 后端项目中，订单、支付、库存这类关键事件不能随便丢。Producer 可靠性配置是第一道防线，但不是全部防线。

---

## 一、消息可能在哪里丢

Producer 发送链路：

```text
Go 业务代码
  -> producer buffer
  -> 网络
  -> broker leader
  -> follower replicas
  -> broker ack
  -> producer 返回
```

可能的问题：

- Go 服务退出时 buffer 里还有消息。
- 网络请求失败。
- broker leader 写入失败。
- leader 写入后还没复制就宕机。
- producer 超时但 broker 实际写入成功。
- 业务没有处理发送失败。

所以可靠性不是一个参数能解决的。

---

## 二、acks 是什么

`acks` 决定 producer 等待 broker 怎样确认。

常见值：

```text
acks=0
acks=1
acks=all
```

它影响：

- 消息丢失概率。
- 发送延迟。
- 吞吐。
- broker 故障时的行为。

---

## 三、acks=0

含义：

```text
producer 不等待 broker 确认
```

Producer 把请求发出去后，就认为完成。

优点：

- 延迟低。
- 吞吐可能高。

缺点：

- broker 没收到也可能不知道。
- 网络失败可能感知不到。
- 丢消息风险高。

适合：

- 极少数可以接受丢失的日志。
- 非关键埋点。

不适合：

- 订单。
- 支付。
- 库存。
- 审计。

---

## 四、acks=1

含义：

```text
partition leader 写入成功后返回 ack
```

优点：

- 比 `acks=0` 可靠。
- 延迟比 `acks=all` 低。

风险：

```text
leader 写入成功后返回 ack
消息还没复制到 follower
leader 宕机
新 leader 没有这条消息
消息可能丢失
```

适合：

- 一些可靠性要求中等的场景。
- 可以通过补偿或重放修复的日志。

---

## 五、acks=all

含义：

```text
leader 等待 ISR 中满足要求的副本确认后，再返回 ack
```

生产关键业务通常使用：

```text
acks=all
```

但要注意，它需要和 broker 配置一起看：

```text
replication.factor=3
min.insync.replicas=2
```

如果 topic 只有一个副本，`acks=all` 也只能等一个副本。

---

## 六、min.insync.replicas

`min.insync.replicas` 表示写入成功所需的最少同步副本数量。

常见生产组合：

```text
replication.factor=3
min.insync.replicas=2
producer acks=all
```

含义：

- 每个 partition 有 3 份副本。
- 至少 2 个副本在 ISR 中时，写入才算成功。
- 如果 ISR 少于 2，producer 写入会失败。

这比“只要 leader 活着就写”更可靠。

---

## 七、retries 是什么

`retries` 控制 producer 发送失败时是否重试。

重试可以处理：

- 临时网络错误。
- broker leader 切换。
- 短暂请求超时。
- broker 短暂不可用。

但重试也带来问题：

```text
可能产生重复消息
可能影响顺序
可能让请求延迟变长
```

---

## 八、重试为什么可能重复

场景：

```text
producer 发送消息
broker 实际写入成功
broker 返回 ack 时网络断了
producer 认为失败
producer 重试
broker 再写一次
```

结果：

```text
Kafka 中可能有两条相同业务消息
```

所以 consumer 必须幂等。

producer retry 不能代替 consumer 幂等。

---

## 九、delivery timeout

很多客户端有整体发送超时配置。

直觉：

```text
一条消息从进入 producer 到最终成功或失败，最多允许多久
```

如果超时太短：

- 正常的短暂抖动也可能失败。

如果超时太长：

- 业务请求可能等待太久。
- 错误暴露太慢。

Go 代码中也应该使用 context 控制：

```go
ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
defer cancel()
```

---

## 十、关键业务推荐配置直觉

订单、支付、库存事件：

```text
acks=all
enable.idempotence=true
retries 开启
delivery timeout 合理
message key 稳定
```

broker/topic：

```text
replication.factor=3
min.insync.replicas=2
监控 under replicated partitions
```

consumer：

```text
业务幂等
手动提交 offset
retry topic
DLQ
```

可靠性是链路整体，不是 producer 单点。

---

## 十一、Go 后端错误处理

发送失败时，不同业务有不同处理：

### 1. 请求必须失败

例如创建订单必须发布事件成功才返回成功。

```go
if err := producer.Publish(ctx, topic, msg); err != nil {
    return fmt.Errorf("publish order event: %w", err)
}
```

### 2. 使用 Outbox

更推荐关键业务使用 outbox：

```text
同一数据库事务写 orders 和 outbox_events
后台 worker 发布 Kafka
失败可重试
```

这样避免：

```text
数据库成功但 Kafka 失败
```

### 3. 非关键日志降级

行为日志发送失败可以记录错误并丢弃或本地缓冲，取决于业务要求。

---

## 十二、命令行能不能完整演示 acks

命令行 producer 可以传一些 producer property，但学习可靠性更重要的是理解配置组合。

真实验证通常要：

- 多 broker 集群。
- 设置 replication factor。
- 模拟 broker 宕机。
- 观察 producer 错误。

本地单节点只能建立直觉，不能完整验证生产高可用。

---

## 十三、常见误区

### 1. acks=all 就永不丢消息

不是。

还要看副本数、ISR、磁盘、unclean leader election、客户端错误处理。

### 2. retries 开启后业务就不会失败

不是。

重试有上限，也可能超时。

### 3. producer 幂等等于业务幂等

不是。

producer 幂等只解决 producer 到 broker 的一部分重复问题。

业务幂等要在 consumer 和数据库层处理。

### 4. 发送失败可以只打印日志

关键业务不行。

必须有补偿、重试、outbox 或明确返回错误。

---

## 十四、本节练习

1. 解释 `acks=0`、`acks=1`、`acks=all` 的区别。
2. 画出 `acks=1` 下 leader 宕机导致消息丢失的场景。
3. 解释 producer retry 为什么可能产生重复消息。
4. 为 `order.created` 设计 producer 可靠性配置。
5. 思考：如果 Kafka 发送失败，创建订单接口应该返回成功还是失败？
6. 解释为什么 outbox pattern 常用于关键业务事件。

---

## 十五、本节小结

- `acks` 决定 producer 等待 broker 怎样确认。
- `acks=0` 风险最高，关键业务不推荐。
- `acks=1` 等 leader 确认，但 leader 崩溃时仍可能丢。
- `acks=all` 更可靠，但要配合副本和 min ISR。
- `retries` 可以处理临时失败，但可能导致重复消息。
- consumer 业务幂等是必须的。
- Go 后端要对发送失败有明确处理。
- 关键业务更推荐结合 outbox pattern。

---

## 十六、订单事件推荐配置

对 `order.created`：

```text
acks=all
retries > 0
enable.idempotence=true
message key=order_id
```

同时要配合 broker/topic：

```text
replication.factor=3
min.insync.replicas=2
```

本地学习环境可能只有一个 broker，所以只能理解配置含义；生产环境才真正体现多副本可靠性。

---

## 十七、发送失败处理决策

如果创建订单时 Kafka 发送失败，有两种选择：

```text
直接返回失败：简单，但用户体验受 Kafka 影响。
使用 Outbox：订单事务成功，事件稍后补发。
```

关键订单链路通常选择 Outbox。

---

## 十八、最终检查

完成本节后，请写出：

```text
acks=all 依赖哪些 broker/topic 配置？
producer retry 为什么不能替代业务幂等？
Kafka 发送失败时，order-service 的策略是什么？
```

这些问题能回答清楚，说明可靠性不是只停留在参数层。

---

## 二十一、参数不是孤立生效的

评估发送可靠性时，至少同时看四个因素：

| 因素 | 关注点 |
| --- | --- |
| `acks` | broker 确认到什么程度才算成功 |
| `retries` | 网络抖动时是否自动重试 |
| `delivery.timeout.ms` | 一条消息最多等待多久 |
| 幂等 Producer | 重试是否可能造成 broker 端重复写入 |

如果只把 `retries` 调大，却不理解超时和幂等，就可能得到“应用以为失败，broker 实际写入成功”的模糊状态。

Go 服务日志必须保留 event_id，这样才能在模糊状态下做人工核对。
