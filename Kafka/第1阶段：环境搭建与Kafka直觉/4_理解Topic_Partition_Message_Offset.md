# 4. 理解 Topic、Partition、Message、Offset

本节目标：理解 Kafka 中最基础的四个对象：topic、partition、message、offset，并能把它们和 Go 后端业务事件联系起来。

如果说 PostgreSQL 的基础对象是 database、schema、table、row、column，那么 Kafka 的基础对象就是 topic、partition、message、offset。后面学习 producer、consumer、consumer group、lag、顺序性，都会反复用到这四个概念。

---

## 一、Topic：消息的逻辑分类

Topic 可以理解为一类消息的名字。

例如：

```text
order.created
payment.succeeded
inventory.deducted
user.registered
```

一个 topic 通常对应一类业务事件。

在 Go 后端项目中，topic 名经常会出现在：

- producer 配置。
- consumer 订阅列表。
- 日志字段。
- 监控标签。
- 告警规则。
- retry topic 和 DLQ 命名。

---

## 二、Topic 命名原则

推荐使用：

```text
领域对象.过去式动作
```

例如：

```text
order.created
order.paid
order.cancelled
payment.failed
coupon.granted
```

这样命名的好处是：

- 清楚表达发生了什么。
- 下游服务读 topic 名就知道语义。
- 适合事件驱动架构。
- 方便设计 retry 和 DLQ。

例如：

```text
order.created.retry.1m
order.created.dlq
```

---

## 三、Partition：Topic 的物理分片

一个 topic 可以有多个 partition。

例如：

```text
topic: order.created
  partition-0
  partition-1
  partition-2
```

partition 解决两个问题：

1. 提高吞吐。
2. 提高消费并行度。

如果所有消息都只有一个 partition，那么同一个 consumer group 中最多只有一个 consumer 能消费这个 topic。

有多个 partition 后，不同 partition 可以分配给不同 consumer。

---

## 四、Message：Kafka 中的一条消息

一条 Kafka 消息通常包含：

- topic。
- partition。
- offset。
- key。
- value。
- headers。
- timestamp。

在业务上，一条消息可以是一个事件。

例如 `order.created`：

```json
{
  "event_id": "evt_order_created_1001",
  "event_type": "order.created",
  "version": 1,
  "occurred_at": "2026-07-05T12:00:00Z",
  "data": {
    "order_id": "order_1001",
    "user_id": "user_88",
    "total_amount": 19800
  }
}
```

字段说明：

- `event_id`：用于幂等。
- `event_type`：用于识别事件类型。
- `version`：用于消息结构演进。
- `occurred_at`：事件发生时间。
- `data`：业务数据。

---

## 五、Key：决定消息去哪一个 Partition

Kafka producer 发送消息时，可以指定 key。

如果指定 key，Kafka 会根据 key 选择 partition。

对于订单事件，常用：

```text
key = order_id
```

这样同一个订单的事件通常会进入同一个 partition。

为什么这很重要？

因为 Kafka 只保证：

```text
同一个 partition 内有序
```

如果 `order.created`、`order.paid`、`order.cancelled` 被分到不同 partition，就不能依赖 Kafka 保证它们的消费顺序。

---

## 六、Offset：Partition 内的位置

offset 是消息在某个 partition 内的位置。

例如：

```text
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

不存在全局 topic offset。

`partition-0 offset=1` 和 `partition-1 offset=1` 是两条不同消息。

---

## 七、Offset 和 Consumer Group 的关系

consumer group 会记录自己在每个 partition 上消费到了哪里。

例如：

```text
group: inventory-service
topic: order.created
partition: 0
committed offset: 100
```

这表示：

```text
inventory-service 这个 group 在 order.created 的 partition 0 上已经提交到了某个位置
```

另一个 group，比如 `notification-service`，有自己的 offset。

所以同一个 topic 可以被多个服务独立消费。

---

## 八、用命令观察这些概念

创建 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --if-not-exists \
  --topic concept.orders \
  --partitions 3 \
  --replication-factor 1
```

查看 topic：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic concept.orders
```

发送带 key 消息：

```bash
kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic concept.orders \
  --property parse.key=true \
  --property key.separator=:
```

输入：

```text
order-1:created
order-1:paid
order-2:created
order-2:paid
```

消费并打印 partition 和 offset：

```bash
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic concept.orders \
  --from-beginning \
  --property print.key=true \
  --property print.partition=true \
  --property print.offset=true
```

观察：

- 相同 key 通常进入同一个 partition。
- 每个 partition 内 offset 递增。
- 不同 partition 的 offset 各自从 0 开始。

---

## 九、Go 后端工程视角

在 Go 代码里，推荐定义统一消息结构：

```go
type Event struct {
    EventID    string          `json:"event_id"`
    EventType  string          `json:"event_type"`
    Version    int             `json:"version"`
    OccurredAt time.Time       `json:"occurred_at"`
    Data       json.RawMessage `json:"data"`
}
```

业务事件再定义自己的 data：

```go
type OrderCreatedData struct {
    OrderID     string `json:"order_id"`
    UserID      string `json:"user_id"`
    TotalAmount int64  `json:"total_amount"`
}
```

producer 发送时：

```text
topic = order.created
key = order_id
value = event JSON
headers = trace_id, event_type, schema_version
```

consumer 日志至少记录：

```text
topic
partition
offset
key
event_id
group
handler
```

这些字段在线上排查时非常关键。

---

## 十、常见误区

### 1. Topic 越多越好

不是。

topic 应该按业务事件分类，不要为每个 consumer 创建一个 topic。

错误：

```text
order.created.for.inventory
order.created.for.notification
```

更好：

```text
order.created
```

然后让不同 consumer group 独立订阅。

### 2. Partition 越多越好

不是。

partition 太多会增加 broker 和 consumer group 的管理开销。

### 3. Offset 是全局递增的

不是。

offset 只在 partition 内递增。

### 4. 同一个 topic 全局有序

不是。

多 partition topic 不保证全局顺序。

---

## 十一、本节练习

1. 为用户注册事件设计 topic 名称。
2. 为支付成功事件设计 topic 名称。
3. 判断订单事件应该用 `order_id` 还是 `user_id` 做 key，并说明理由。
4. 创建一个 3 partition 的 topic。
5. 发送 10 条带 key 的消息。
6. 打印 partition 和 offset。
7. 记录相同 key 是否进入同一 partition。
8. 用自己的话解释：为什么 offset 不是全局唯一位置？

---

## 十二、本节小结

- topic 是消息的逻辑分类。
- partition 是 topic 的物理分片。
- message 是 Kafka 中的一条记录。
- key 通常决定消息进入哪个 partition。
- Kafka 只保证同一个 partition 内有序。
- offset 是消息在 partition 内的位置。
- offset 不属于 topic 全局，而属于 partition。
- consumer group 会记录自己在每个 partition 上的消费进度。
- Go 后端日志必须记录 topic、partition、offset、key 和 event_id。

---

## 二十一、用一张表复盘

学习完本节后，建议把一次消费记录整理成下面的表格：

| 字段 | 示例 | 说明 |
| --- | --- | --- |
| topic | `order.created` | 业务事件流名称 |
| partition | `2` | 消息所在分区 |
| offset | `158` | 该分区内的位置 |
| key | `order_1001` | 决定分区选择和局部顺序 |
| group | `inventory-service` | 消费进度归属 |

这张表能帮助你避免一个常见误解：offset 不是消息的全局编号，而是某个 partition 内部的递增位置。

面试或排障时，如果你能先说清这张表，再继续讨论重试、幂等和 lag，表达会更稳。
