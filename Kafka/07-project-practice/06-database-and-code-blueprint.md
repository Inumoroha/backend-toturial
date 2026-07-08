# 06 数据库与代码蓝图

本节目标：把项目从“架构设计”推进到“可以开始编码”。这里给出表结构、接口、模块边界和关键流程伪代码。

## 数据库表设计

学习项目可以先用 PostgreSQL。MySQL 也可以，字段类型按需调整。

### orders

```sql
CREATE TABLE orders (
  id VARCHAR(64) PRIMARY KEY,
  user_id VARCHAR(64) NOT NULL,
  status VARCHAR(32) NOT NULL,
  total_amount BIGINT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

订单状态建议：

```text
CREATED
INVENTORY_DEDUCTED
INVENTORY_FAILED
PAID
CANCELLED
```

### order_items

```sql
CREATE TABLE order_items (
  id BIGSERIAL PRIMARY KEY,
  order_id VARCHAR(64) NOT NULL,
  sku_id VARCHAR(64) NOT NULL,
  quantity INT NOT NULL,
  price BIGINT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### inventory

```sql
CREATE TABLE inventory (
  sku_id VARCHAR(64) PRIMARY KEY,
  available INT NOT NULL,
  locked INT NOT NULL DEFAULT 0,
  updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### inventory_logs

```sql
CREATE TABLE inventory_logs (
  id BIGSERIAL PRIMARY KEY,
  event_id VARCHAR(128) NOT NULL,
  order_id VARCHAR(64) NOT NULL,
  sku_id VARCHAR(64) NOT NULL,
  quantity INT NOT NULL,
  action VARCHAR(32) NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### processed_events

```sql
CREATE TABLE processed_events (
  event_id VARCHAR(128) PRIMARY KEY,
  topic VARCHAR(128) NOT NULL,
  partition_id INT NOT NULL,
  offset_value BIGINT NOT NULL,
  handler VARCHAR(128) NOT NULL,
  processed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### outbox_events

```sql
CREATE TABLE outbox_events (
  id VARCHAR(128) PRIMARY KEY,
  aggregate_type VARCHAR(64) NOT NULL,
  aggregate_id VARCHAR(128) NOT NULL,
  event_type VARCHAR(128) NOT NULL,
  topic VARCHAR(128) NOT NULL,
  message_key VARCHAR(128) NOT NULL,
  payload JSONB NOT NULL,
  status VARCHAR(32) NOT NULL,
  retry_count INT NOT NULL DEFAULT 0,
  next_retry_at TIMESTAMP NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  sent_at TIMESTAMP NULL
);

CREATE INDEX idx_outbox_status_next_retry
ON outbox_events(status, next_retry_at);
```

状态：

```text
PENDING
SENDING
SENT
FAILED
```

## HTTP API

### 创建订单

```http
POST /orders
Content-Type: application/json

{
  "user_id": "user_88",
  "items": [
    {
      "sku_id": "sku_1",
      "quantity": 2,
      "price": 9900
    }
  ]
}
```

响应：

```json
{
  "order_id": "order_1001",
  "status": "CREATED"
}
```

## 创建订单流程

伪代码：

```go
func CreateOrder(ctx context.Context, req CreateOrderRequest) error {
    tx := db.BeginTx(ctx)
    defer tx.Rollback()

    orderID := newOrderID()
    eventID := "evt_order_created_" + orderID

    insertOrder(tx, orderID, req)
    insertOrderItems(tx, orderID, req.Items)

    event := BuildOrderCreatedEvent(eventID, orderID, req)
    insertOutbox(tx, OutboxEvent{
        ID:            eventID,
        AggregateType: "order",
        AggregateID:   orderID,
        EventType:     "order.created",
        Topic:         "order.created",
        MessageKey:    orderID,
        Payload:       event,
        Status:        "PENDING",
    })

    return tx.Commit()
}
```

关键点：

- 订单和 outbox 必须在同一个数据库事务中。
- event id 稳定，不能每次重试生成新的。
- HTTP 请求不一定直接发送 Kafka，可以让 outbox worker 发送。

## Outbox Worker 流程

伪代码：

```go
func RunOutboxWorker(ctx context.Context) {
    for ctx.Err() == nil {
        events := lockPendingOutboxEvents(ctx, 100)

        for _, event := range events {
            err := producer.Publish(ctx, event.Topic, event.MessageKey, event.Payload)
            if err != nil {
                markOutboxRetry(ctx, event.ID, err)
                continue
            }

            markOutboxSent(ctx, event.ID)
        }

        sleepOrWait(ctx, time.Second)
    }
}
```

注意：

- 多 worker 并发时要避免重复抢同一条 outbox。
- PostgreSQL 可以用 `FOR UPDATE SKIP LOCKED`。
- 发送成功但标记 sent 失败时，后续可能重复发送，所以 consumer 必须幂等。

## Inventory Consumer 流程

伪代码：

```go
func HandleOrderCreated(ctx context.Context, msg kafka.Message) error {
    event, err := DecodeOrderCreated(msg.Value)
    if err != nil {
        return NonRetryable(err)
    }

    tx := db.BeginTx(ctx)
    defer tx.Rollback()

    inserted, err := insertProcessedEvent(tx, event.EventID, msg)
    if err != nil {
        return Retryable(err)
    }
    if !inserted {
        return tx.Commit()
    }

    for _, item := range event.Data.Items {
        ok, err := deductInventory(tx, item.SkuID, item.Quantity)
        if err != nil {
            return Retryable(err)
        }
        if !ok {
            insertInventoryFailedOutbox(tx, event, item)
            return tx.Commit()
        }
    }

    insertInventoryDeductedOutbox(tx, event)
    return tx.Commit()
}
```

consumer wrapper 收到结果后：

```text
handler success -> commit offset
retryable error -> write retry topic -> commit offset
non-retryable error -> write DLQ -> commit offset
```

## 代码模块边界

```text
internal/kafka
  只处理 Kafka 通用能力

internal/order
  创建订单、订单表、订单事件、outbox 写入

internal/inventory
  库存扣减、processed_events、库存事件

internal/outbox
  扫描 outbox_events 并发布 Kafka

internal/platform
  日志、配置、数据库、shutdown、metrics
```

不要让 `internal/kafka` 依赖 `internal/order`。依赖方向应该是业务模块调用基础设施，而不是基础设施知道业务。

## 服务启动顺序

1. 启动 Kafka 和数据库。
2. 执行 migration。
3. 创建 topic。
4. 启动 outbox worker。
5. 启动 inventory-service consumer。
6. 启动 order-service API。
7. 调用创建订单 API。
8. 观察事件流转。

## Topic 创建脚本

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --create --if-not-exists --topic order.created --partitions 3 --replication-factor 1
kafka-topics.sh --bootstrap-server localhost:9092 --create --if-not-exists --topic order.created.retry.1m --partitions 3 --replication-factor 1
kafka-topics.sh --bootstrap-server localhost:9092 --create --if-not-exists --topic order.created.dlq --partitions 3 --replication-factor 1
kafka-topics.sh --bootstrap-server localhost:9092 --create --if-not-exists --topic inventory.deducted --partitions 3 --replication-factor 1
kafka-topics.sh --bootstrap-server localhost:9092 --create --if-not-exists --topic inventory.failed --partitions 3 --replication-factor 1
```

## 本节验收

你需要能完成：

1. 根据表结构创建数据库 migration。
2. 写出创建订单时插入 outbox 的代码。
3. 写出 outbox worker 的主循环。
4. 写出库存 consumer 的幂等扣减逻辑。
5. 解释为什么 outbox 和 consumer 幂等必须同时存在。

