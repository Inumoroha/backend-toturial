# 5. 幂等 Producer 与顺序性

本节目标：理解 Kafka 幂等 producer 解决什么问题、不能解决什么问题，以及 producer 重试、并发请求和消息顺序之间的关系。

Kafka 中“幂等”这个词很容易被误解。Producer 幂等不是业务幂等。它主要解决 producer 到 broker 写入过程中，因为重试导致的重复写入问题。业务是否重复扣库存、重复发券，仍然要靠 consumer 和数据库设计。

---

## 一、Producer 重试为什么会重复

场景：

```text
producer 发送消息 M
broker 成功写入 M
broker 返回 ack 时网络断开
producer 没收到 ack
producer 认为发送失败
producer 重试 M
broker 再写入一次 M
```

结果：

```text
Kafka 中出现两条相同业务消息
```

这就是 producer 重试可能导致重复的原因。

---

## 二、幂等 Producer 解决什么

幂等 producer 会让 broker 能识别同一个 producer 会话中的重复发送。

直觉上可以理解为 producer 给消息加上：

```text
producer id
sequence number
```

broker 可以判断：

```text
这条是不是已经写过
```

从而避免某些重试导致的重复写入。

---

## 三、幂等 Producer 不解决什么

它不解决：

- consumer 重复消费。
- 业务接口重复调用。
- outbox worker 重复发布。
- 人工重放 DLQ。
- 服务重启后业务事件重新生成。
- 下游数据库重复扣减。

所以不要说：

```text
我开启 producer 幂等，所以业务不会重复
```

更准确：

```text
producer 幂等降低 producer 重试造成的 Kafka 写入重复；业务幂等仍然要在 consumer 侧实现。
```

---

## 四、顺序性边界

Kafka 的顺序性包括几个层次：

### 1. Partition 内顺序

Kafka 保证同一个 partition 内消息有序。

### 2. Topic 全局不保证顺序

多 partition topic 不保证全局顺序。

### 3. 同一 key 的顺序依赖 key 到 partition 的稳定映射

如果同一 key 进入同一 partition，就能获得这个 key 维度的顺序。

---

## 五、重试和顺序的关系

Producer 发送请求可能并发进行。

如果多个请求 in-flight，再加上重试，可能出现顺序风险。

例如：

```text
发送 batch-1
发送 batch-2
batch-1 失败并重试
batch-2 先成功
batch-1 后成功
```

如果配置不当，可能影响顺序。

Kafka 的幂等 producer 和相关配置可以降低这种风险。

---

## 六、max.in.flight.requests.per.connection

这个参数控制同一个连接上最多有多少未完成请求。

值越大：

- 吞吐可能更高。
- 顺序风险可能增加。

值越小：

- 顺序更容易控制。
- 吞吐可能下降。

现代 Kafka 在启用幂等 producer 后，对顺序有更好的保护，但你仍然要理解这个参数背后的取舍。

---

## 七、业务顺序怎么保证

如果你要保证同一订单事件顺序：

1. 使用同一个 topic 或明确的状态事件流。
2. 使用 `order_id` 作为 key。
3. 确保同一订单事件进入同一 partition。
4. consumer 对 partition 内消息顺序处理。
5. consumer 不要随意并发处理同一个 partition 内的消息。

只靠 producer 不够。

---

## 八、乱序时业务如何兜底

真实系统中，完全依赖消息顺序并不总是稳妥。

例如 consumer 先收到：

```text
payment.succeeded
```

后收到：

```text
order.created
```

业务可以通过状态机兜底：

- 如果订单不存在，暂存或重试。
- 如果状态不允许，进入异常流程。
- 使用事件版本或 occurred_at 判断。

对于关键业务，建议状态机不要只依赖消息到达顺序。

---

## 九、Go 后端设计建议

Producer 侧：

```text
关键业务开启幂等 producer
使用稳定 message key
设置合理 retries
记录 event_id
```

Consumer 侧：

```text
手动提交 offset
业务幂等
按 partition 顺序处理
状态机校验
异常进入 retry 或 DLQ
```

数据库侧：

```text
唯一约束
processed_events
业务状态版本
事务
```

---

## 十、示例：订单事件顺序

事件：

```text
order.created key=order_1001
order.paid key=order_1001
order.shipped key=order_1001
```

它们进入同一个 partition。

consumer 顺序处理：

```text
CREATED -> PAID -> SHIPPED
```

如果重复收到 `order.paid`：

```text
订单已是 PAID 或 SHIPPED，直接幂等返回
```

如果先收到 `order.shipped`：

```text
状态机发现当前状态不允许，进入 retry 或异常处理
```

---

## 十一、常见误区

### 1. Producer 幂等等于 Exactly Once 业务语义

不是。

producer 幂等只是 Kafka 写入侧能力。

### 2. 设置 key 后全局有序

不是。

同一 key 通常有序，不同 key 之间不保证顺序。

### 3. Consumer 并发越高越好

不一定。

同一 partition 内并发处理会让 offset 提交和顺序都变复杂。

### 4. 只要 Kafka 有序，业务就不用状态机

不是。

重试、补偿、人工重放、跨 topic 流程都可能带来乱序，业务仍要兜底。

---

## 十二、本节练习

1. 解释 producer 重试为什么可能导致重复写入。
2. 说明 producer 幂等解决什么问题。
3. 说明 producer 幂等不解决什么问题。
4. 为订单事件设计顺序保证方案。
5. 思考 consumer 并发处理同一 partition 会带来什么问题。
6. 设计订单状态机如何处理重复 `payment.succeeded`。

---

## 十三、本节小结

- Producer retry 可能导致重复写入。
- 幂等 producer 可以降低 producer 重试导致的 Kafka 写入重复。
- Producer 幂等不是业务幂等。
- Kafka 只保证 partition 内顺序。
- 同一业务 key 进入同一 partition 才能获得 key 维度顺序。
- 业务顺序还需要 consumer 顺序处理和状态机兜底。
- Go 后端必须同时设计 producer、consumer 和数据库层的可靠性。

---

## 十四、Go Producer 配置方向

不同 Go 客户端配置名不一样，但核心目标一致：

```text
启用幂等 producer。
开启重试。
限制可能破坏顺序的并发发送。
等待 broker 确认。
```

伪配置可以理解为：

```go
ProducerConfig{
    EnableIdempotence: true,
    RequiredAcks:      "all",
    Retries:           10,
    MaxInFlight:       5,
}
```

学习时不要死背字段名，要理解每个参数解决的问题。

---

## 十五、顺序性实验

做一个实验：

```text
topic: order.events
partitions: 3
key: order_id
```

连续发送：

```text
order_1: order.created
order_1: payment.succeeded
order_1: order.completed
```

消费时观察同一个 `order_1` 是否落到同一个 partition。

再把 key 改成随机值，观察事件是否散落到不同 partition。

这个实验会让你直观看到：

```text
Kafka 的顺序性不是 topic 全局顺序，而是 partition 内顺序。
```

---

## 十六、面试表达

可以这样回答：

```text
Kafka producer retry 可能因为 ack 丢失导致重复写入，幂等 producer 能在 producer 会话和分区维度降低这类重复。
但它不能替代业务幂等，因为 outbox 重发、consumer offset 提交失败、DLQ 重放仍然会让同一个业务事件被处理多次。
顺序性方面，Kafka 只保证同一 partition 内有序，所以订单类事件要用 order_id 做 key，并且 consumer 侧也要避免同一订单并发乱序处理。
```

---

## 十七、顺序性设计检查

设计订单事件时检查：

- [ ] 同一订单事件使用同一个 `order_id` 作为 key。
- [ ] consumer 不并发处理同一 partition 中的同一订单。
- [ ] retry 后的乱序风险有状态机兜底。
- [ ] 重放 DLQ 时不会让订单状态倒退。

Kafka 的分区顺序只能提供基础保障，业务状态机仍然需要兜底。

---

## 十八、实验结论模板

```text
使用 order_id 作为 key 时，同一订单事件进入同一个 partition。
使用随机 key 时，同一订单事件可能进入不同 partition。
因此订单维度的顺序要求，必须从 key 设计开始。
```

---

## 十九、最终检查

解释清楚这句话：

```text
Kafka 的顺序性边界是 partition，不是 topic。
```

如果这句话说不清，就不要急着设计订单状态流转。

---

## 二十、状态机兜底示例

即使 Kafka 保证分区内顺序，业务仍要防止状态倒退：

```text
CREATED -> PAID -> SHIPPED
```

如果订单已经是 `SHIPPED`，又收到旧的 `PAID` 事件，不能把状态改回去。

顺序性和状态机要一起设计。

---

## 二十一、故障案例

```text
order.created 和 order.cancelled 使用随机 key。
两个事件进入不同 partition。
consumer 先处理 cancelled，后处理 created。
如果没有状态机校验，订单可能回到 CREATED。
```

这个案例说明 key 设计和状态机都不能省。

---

## 二十二、最终练习

设计一个订单状态机，说明重复或乱序收到下面事件时如何处理：

```text
order.created
payment.succeeded
order.cancelled
```

要求：不能让订单状态倒退。

---

## 二十一、顺序性检查清单

设计订单事件流时，至少检查：

```text
同一订单是否使用同一个 message key。
同一订单事件是否只要求 partition 内局部顺序。
Producer 是否可能并发发送同一订单的不同状态。
Consumer 是否按事件版本号或状态机防止倒退。
重试是否会改变消息顺序。
```

Kafka 只能帮助你维护同一 partition 内的追加顺序，不能替你判断业务状态是否允许从 PAID 回到 CREATED。

因此顺序性通常要由 message key、幂等 Producer 和业务状态机共同保证。
