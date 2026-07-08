# 2. Topic 与事件结构设计

本节目标：设计项目中的 topic、message key、事件 JSON 和 retry/DLQ topic。

---

## 一、Topic 列表

```text
order.created
order.created.retry.1m
order.created.retry.5m
order.created.dlq
inventory.deducted
inventory.failed
```

---

## 二、Message Key

| Topic | Key |
| --- | --- |
| order.created | order_id |
| inventory.deducted | order_id |
| inventory.failed | order_id |

---

## 三、OrderCreatedEvent

```json
{
  "event_id": "evt_order_created_order_1001",
  "event_type": "order.created",
  "version": 1,
  "occurred_at": "2026-07-05T12:00:00Z",
  "data": {
    "order_id": "order_1001",
    "user_id": "user_88",
    "items": [
      {
        "sku_id": "sku_1",
        "quantity": 2,
        "price": 9900
      }
    ],
    "total_amount": 19800
  }
}
```

---

## 四、Go 结构

```go
type OrderCreatedEvent struct {
    EventID    string    `json:"event_id"`
    EventType  string    `json:"event_type"`
    Version    int       `json:"version"`
    OccurredAt time.Time `json:"occurred_at"`
    Data       OrderData `json:"data"`
}
```

---

## 五、设计原则

- 每条事件必须有 event_id。
- 金额用整数分。
- 时间用 UTC。
- 不放敏感字段。
- version 必须存在。

---

## 六、InventoryDeductedEvent

```json
{
  "event_id": "evt_inventory_deducted_order_1001",
  "event_type": "inventory.deducted",
  "version": 1,
  "occurred_at": "2026-07-05T12:00:03Z",
  "data": {
    "order_id": "order_1001",
    "items": [
      {
        "sku_id": "sku_1",
        "quantity": 2
      }
    ]
  }
}
```

key：

```text
order_id
```

---

## 七、InventoryFailedEvent

```json
{
  "event_id": "evt_inventory_failed_order_1001",
  "event_type": "inventory.failed",
  "version": 1,
  "occurred_at": "2026-07-05T12:00:03Z",
  "data": {
    "order_id": "order_1001",
    "reason": "insufficient_inventory",
    "items": [
      {
        "sku_id": "sku_1",
        "required": 2,
        "available": 1
      }
    ]
  }
}
```

库存不足是业务结果，不应该无限 retry。

---

## 八、Retry Message

```json
{
  "event_id": "evt_order_created_order_1001",
  "original_topic": "order.created",
  "original_partition": 1,
  "original_offset": 203,
  "original_key": "order_1001",
  "attempt": 1,
  "last_error": "database timeout",
  "next_retry_at": "2026-07-05T12:01:00Z",
  "payload": {}
}
```

---

## 九、DLQ Message

```json
{
  "original_topic": "order.created",
  "original_partition": 1,
  "original_offset": 203,
  "original_key": "order_1001",
  "error_type": "invalid_json",
  "error_message": "unexpected end of JSON input",
  "failed_at": "2026-07-05T12:00:00Z",
  "payload": "bad-json"
}
```

---

## 十、Go 文件建议

```text
internal/order/event.go
internal/inventory/event.go
internal/kafka/retry.go
internal/kafka/dlq.go
```

每个 event struct 要和 JSON 示例保持一致。

---

## 十一、Topic 创建脚本

`deployments/topics.sh`：

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --if-not-exists --topic order.created --partitions 3 --replication-factor 1
kafka-topics.sh --bootstrap-server localhost:9092 --create --if-not-exists --topic order.created.retry.1m --partitions 3 --replication-factor 1
kafka-topics.sh --bootstrap-server localhost:9092 --create --if-not-exists --topic order.created.dlq --partitions 3 --replication-factor 1
kafka-topics.sh --bootstrap-server localhost:9092 --create --if-not-exists --topic inventory.deducted --partitions 3 --replication-factor 1
kafka-topics.sh --bootstrap-server localhost:9092 --create --if-not-exists --topic inventory.failed --partitions 3 --replication-factor 1
```

---

## 十二、本节练习

1. 写出 `OrderCreatedEvent` Go struct。
2. 写出 `InventoryDeductedEvent` Go struct。
3. 写出 retry message struct。
4. 写出 DLQ message struct。
5. 解释为什么 `order_id` 适合作为 key。

---

## 十三、Topic 参数怎么选

学习环境里可以统一使用：

```text
partitions = 3
replication-factor = 1
```

原因：

```text
本地 Docker 通常只有一个 broker，所以副本数只能是 1。
分区数设为 3，可以练习 key 分区、consumer group 和 lag。
```

生产环境不能照抄。生产里至少要考虑：

| 参数 | 学习环境 | 生产常见思路 |
| --- | --- | --- |
| partitions | 3 | 按吞吐、并发 consumer 数、未来扩容估算 |
| replication-factor | 1 | 通常 3 |
| min.insync.replicas | 1 | 通常 2 |
| retention.ms | 默认 | 按重放需求设置 |

先记住一句话：

```text
分区数影响并发和顺序边界，副本数影响可用性。
```

---

## 十四、为什么 order.created 用 order_id 做 key

Kafka 同一个 key 会被写入同一个分区。

对订单事件来说，我们希望同一订单相关事件尽量有序：

```text
order.created
inventory.deducted
payment.succeeded
order.completed
```

如果它们都用 `order_id` 作为 key，那么同一个订单的同类事件在对应 topic 内会进入固定分区。

不要用随机 key：

```text
随机 key 会让同一订单事件散落到不同分区。
```

也不要简单使用空 key：

```text
空 key 会让消息轮询分布，无法利用业务维度的顺序性。
```

---

## 十五、事件命名规范

建议使用过去式：

```text
order.created
inventory.deducted
inventory.failed
payment.succeeded
```

原因是事件表示“已经发生的事实”，不是命令。

不要写成：

```text
create.order
deduct.inventory
send.notification
```

这些更像命令，会让 consumer 误以为自己是在执行上游的远程调用。事件驱动里，上游只声明事实，下游自己决定如何响应。

---

## 十六、事件版本演进示例

第一版事件：

```json
{
  "event_type": "order.created",
  "version": 1,
  "data": {
    "order_id": "order_1001",
    "user_id": "user_88"
  }
}
```

第二版新增 `coupon_id`：

```json
{
  "event_type": "order.created",
  "version": 2,
  "data": {
    "order_id": "order_1001",
    "user_id": "user_88",
    "coupon_id": "coupon_9"
  }
}
```

兼容原则：

- 新字段尽量可选。
- 不随便删除旧字段。
- 不改变旧字段含义。
- consumer 遇到未知字段应该忽略。

Go 里可以这样保留兼容：

```go
type OrderData struct {
    OrderID  string `json:"order_id"`
    UserID   string `json:"user_id"`
    CouponID string `json:"coupon_id,omitempty"`
}
```

---

## 十七、Topic 创建后验证

创建 topic 后检查：

```bash
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic order.created
```

你应该看到类似输出：

```text
Topic: order.created
PartitionCount: 3
ReplicationFactor: 1
```

再列出所有 topic：

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --list
```

确认至少包含：

```text
order.created
order.created.retry.1m
order.created.retry.5m
order.created.dlq
inventory.deducted
inventory.failed
```

---

## 十八、事件结构校验函数

consumer 不应该拿到 JSON 就直接处理，至少先校验基础字段：

```go
func (e OrderCreatedEvent) Validate() error {
    if e.EventID == "" {
        return errors.New("event_id is required")
    }
    if e.EventType != "order.created" {
        return fmt.Errorf("unexpected event_type: %s", e.EventType)
    }
    if e.Version != 1 {
        return fmt.Errorf("unsupported version: %d", e.Version)
    }
    if e.Data.OrderID == "" {
        return errors.New("data.order_id is required")
    }
    if len(e.Data.Items) == 0 {
        return errors.New("data.items is required")
    }
    return nil
}
```

校验失败一般属于不可重试错误，应该进入 DLQ。

---

## 十九、常见错误

### 1. 没有 event_id

没有 `event_id`，consumer 很难做业务幂等。

### 2. 把数据库自增 id 当事件 id

事件 id 最好表达业务语义，例如 `evt_order_created_order_1001`。自增 id 在跨库、重放、排查时可读性差。

### 3. retry topic 和主 topic 分区数不一致

学习阶段最好保持一致，避免重试后 key 分布和消费并发变得难以解释。

### 4. DLQ 不记录原始 offset

没有 `original_partition` 和 `original_offset`，排查时很难定位原消息。
