# 2. Message Key 与 Partition 选择

本节目标：理解 Kafka producer 如何根据 message key 选择 partition，并学会在 Go 后端业务中为不同事件选择合适的 key。

上一阶段已经知道 partition 是并行和顺序的边界。这一节从 producer 视角看：消息到底为什么会进入某个 partition？我们能控制什么？key 选错会造成什么问题？

---

## 一、为什么 Key 很重要

Kafka 的顺序保证范围是：

```text
同一个 partition 内有序
```

如果你希望同一个订单的事件按顺序被消费，就需要让它们进入同一个 partition。

例如：

```text
order.created
order.paid
order.shipped
```

如果这三条消息都用：

```text
key = order_id
```

它们通常会进入同一个 partition。

---

## 二、没有 Key 会怎样

如果不设置 key，客户端会用默认策略选择 partition。

不同客户端策略可能不同，例如：

- 轮询。
- 随机。
- sticky partition。

无 key 的好处：

- 分布可能更均匀。
- 适合不关心顺序的日志。

坏处：

- 同一业务对象的消息可能进入不同 partition。
- 不能依赖业务实体维度顺序。

---

## 三、有 Key 会怎样

有 key 时，producer 通常会基于 key 计算 partition。

直觉：

```text
partition = hash(key) % partition_count
```

这意味着：

```text
相同 key 在 partition 数不变时通常进入同一个 partition
```

注意“通常”是因为不同客户端、不同分区器实现可能有细节差异，但工程直觉就是这样。

---

## 四、订单事件如何选择 Key

订单事件通常使用：

```text
key = order_id
```

原因：

- 同一订单事件需要顺序。
- `order_id` 分布通常较均匀。
- 日志排查方便。
- 下游可以按订单维度幂等。

示例：

```text
topic=order.created key=order_1001
topic=order.paid key=order_1001
topic=order.cancelled key=order_1001
```

---

## 五、支付事件如何选择 Key

支付事件可以选择：

```text
key = order_id
```

或：

```text
key = payment_id
```

如何判断？

如果下游主要按订单聚合流程，例如订单状态机：

```text
key = order_id
```

如果下游主要按支付单处理，例如支付流水对账：

```text
key = payment_id
```

不要机械套模板。key 要服务业务顺序和消费模型。

---

## 六、库存事件如何选择 Key

库存事件更复杂。

可能选择：

```text
key = sku_id
```

好处：

- 同一个商品库存变化有序。
- 适合商品维度库存状态同步。

也可能选择：

```text
key = order_id
```

好处：

- 和订单流程保持一致。
- 适合订单驱动的库存扣减事件。

如果一个订单包含多个 sku，要看下游到底按什么维度处理。

---

## 七、热点 Key

热点 key 是 producer key 设计中最常见的问题之一。

例如：

```text
key = "all"
```

或者某个超级用户、超级商家、爆款 sku 产生大量消息。

结果：

- 大量消息进入同一个 partition。
- 单个 partition lag 飙升。
- 增加 consumer 数量也没用。
- 处理延迟变高。

解决思路：

- 重新评估是否必须强顺序。
- 拆分 key，例如 `sku_id + shard_id`。
- 对热点业务单独建 topic。
- 使用业务层聚合和补偿。

---

## 八、Key 与幂等不是一回事

key 用于分区和局部顺序。

幂等通常依赖：

```text
event_id
business_id
unique constraint
processed_events
```

不要以为：

```text
key 相同就不会重复消费
```

Kafka 仍然可能重复投递。

consumer 仍然必须做幂等。

---

## 九、命令实验：观察 Key 分布

创建 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --if-not-exists \
  --topic stage3.key.demo \
  --partitions 3 \
  --replication-factor 1
```

启动 producer：

```bash
kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic stage3.key.demo \
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
  --topic stage3.key.demo \
  --from-beginning \
  --property print.key=true \
  --property print.partition=true \
  --property print.offset=true
```

记录相同 key 的 partition。

---

## 十、Go 代码示例

发送订单事件：

```go
record := &kgo.Record{
    Topic: "order.created",
    Key:   []byte(orderID),
    Value: payload,
    Headers: []kgo.RecordHeader{
        {Key: "event_id", Value: []byte(eventID)},
        {Key: "event_type", Value: []byte("order.created")},
    },
}
```

发送用户行为日志，如果不要求用户维度顺序，可以不设置 key：

```go
record := &kgo.Record{
    Topic: "user.behavior",
    Value: payload,
}
```

但如果你希望同一用户行为有序：

```go
Key: []byte(userID)
```

---

## 十一、Key 设计检查表

设计 key 时检查：

- 是否需要顺序？
- 顺序是哪个业务维度？
- key 是否分布均匀？
- 是否可能出现热点 key？
- 增加 partition 后是否影响顺序假设？
- 日志排查时 key 是否有用？

---

## 十二、常见误区

### 1. 所有 topic 都必须设置 key

不是。

日志类、不关心顺序的消息可以不设置 key。

### 2. key 用 event_id 最安全

不一定。

event_id 通常每条消息都不同，用它做 key 会让同一订单事件分散到不同 partition。

### 3. key 相同就保证业务只执行一次

不是。

key 不负责幂等。

### 4. partition 数变化不影响 key 映射

通常会影响。

增加 partition 前要评估顺序性。

---

## 十三、本节练习

1. 为 `order.created` 选择 key，并说明理由。
2. 为 `payment.succeeded` 选择 key，并说明理由。
3. 为 `inventory.changed` 选择 key，并说明理由。
4. 设计一个可能产生热点 key 的场景。
5. 写出解决热点 key 的两种思路。
6. 用命令实验观察相同 key 是否进入同一 partition。

---

## 十四、本节小结

- key 影响 producer 选择 partition。
- 同一 key 通常进入同一 partition。
- key 是 Kafka 顺序性的关键工具。
- 订单事件通常用 `order_id` 做 key。
- key 设计要兼顾顺序和分布均匀。
- 热点 key 会导致单 partition lag 高。
- key 不等于幂等键。
- Go producer 应显式设置关键业务事件 key。

---

## 十五、Key 选择表

| 事件 | 推荐 key | 原因 |
| --- | --- | --- |
| `order.created` | `order_id` | 保证订单维度顺序 |
| `payment.succeeded` | `order_id` 或 `payment_id` | 看下游按订单还是支付处理 |
| `inventory.changed` | `sku_id` | 保证商品库存维度顺序 |
| `user.profile.updated` | `user_id` | 用户维度状态更新 |

key 的选择要围绕“哪个业务对象需要顺序”。

---

## 十六、热点 key 处理

如果某个 key 特别热，可以考虑：

- 拆分业务 key，例如 `sku_id + shard_id`。
- 拆分 topic。
- 把热点业务单独处理。
- 优化该 partition 对应 consumer 的 handler。

不要为了均匀分布随便改成随机 key，否则可能破坏顺序。

---

## 十七、Key 决策说明模板

```markdown
## Message Key 设计

事件：order.created
候选 key：order_id、user_id、random
最终选择：order_id
原因：
- 同一订单事件需要顺序。
- 下游库存和订单状态处理都围绕 order_id。
- user_id 可能造成大客户热点。
- random 会破坏订单维度顺序。
```

写 key 设计时，把放弃的选项也说明清楚。

---

## 二十一、key 设计交付模板

为一个 topic 设计 key 时，建议写成下面这样：

```text
topic：order.created
候选 key：order_id、user_id、shop_id、空 key
最终选择：order_id
选择理由：同一订单的事件需要保持分区内顺序
放弃 user_id 的原因：大用户可能导致热点
放弃空 key 的原因：无法保证订单维度顺序
需要观察的指标：各 partition 消息量是否均衡
```

这个模板能让 key 设计变成可评审的工程决策，而不是靠感觉选择一个字段。
