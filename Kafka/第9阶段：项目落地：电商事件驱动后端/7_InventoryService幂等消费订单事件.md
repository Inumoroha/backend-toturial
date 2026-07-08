# 7. InventoryService 幂等消费订单事件

本节目标：实现库存服务消费 `order.created`，使用 processed_events 防止重复扣库存。

---

## 一、消费流程

```text
拉取 order.created
解析事件
begin tx
insert processed_events
如果重复 -> commit -> commit offset
扣减库存
写库存流水
commit
commit offset
```

---

## 二、幂等插入

```sql
INSERT INTO processed_events(event_id, topic, partition_id, offset_value, handler)
VALUES ($1, $2, $3, $4, $5)
ON CONFLICT (event_id) DO NOTHING;
```

如果影响行数为 0，说明处理过。

---

## 三、扣库存 SQL

```sql
UPDATE inventory
SET available = available - $1,
    updated_at = now()
WHERE sku_id = $2
  AND available >= $1;
```

根据影响行数判断库存是否足够。

---

## 四、错误分类

- JSON 错误：不可重试，DLQ。
- 数据库超时：可重试，retry。
- 库存不足：业务失败事件，不是系统错误。

---

## 五、验收

- 同一 event_id 消费两次，库存只扣一次。
- 数据库错误进入 retry。
- 非法消息进入 DLQ。

---

## 六、事件结构

`internal/inventory/event.go`：

```go
type OrderCreatedEvent struct {
    EventID   string    `json:"event_id"`
    EventType string    `json:"event_type"`
    Version   int       `json:"version"`
    Data      OrderData `json:"data"`
}

type OrderData struct {
    OrderID string      `json:"order_id"`
    UserID  string      `json:"user_id"`
    Items   []OrderItem `json:"items"`
}

type OrderItem struct {
    SkuID    string `json:"sku_id"`
    Quantity int    `json:"quantity"`
}
```

---

## 七、Handler

```go
type OrderCreatedHandler struct {
    service *Service
}

func (h *OrderCreatedHandler) Handle(ctx context.Context, msg kafka.ConsumedMessage) error {
    var event OrderCreatedEvent
    if err := json.Unmarshal(msg.Value, &event); err != nil {
        return kafka.NonRetryable(err)
    }

    if event.EventID == "" || event.Data.OrderID == "" {
        return kafka.NonRetryable(errors.New("invalid order.created event"))
    }

    return h.service.Deduct(ctx, msg, event)
}
```

---

## 八、Service 扣库存

```go
func (s *Service) Deduct(ctx context.Context, msg kafka.ConsumedMessage, event OrderCreatedEvent) error {
    tx, err := s.pool.Begin(ctx)
    if err != nil {
        return kafka.Retryable(err)
    }
    defer tx.Rollback(ctx)

    inserted, err := s.repo.InsertProcessedEvent(ctx, tx, event.EventID, msg)
    if err != nil {
        return kafka.Retryable(err)
    }
    if !inserted {
        return tx.Commit(ctx)
    }

    for _, item := range event.Data.Items {
        ok, err := s.repo.DeductInventory(ctx, tx, item.SkuID, item.Quantity)
        if err != nil {
            return kafka.Retryable(err)
        }
        if !ok {
            return s.repo.InsertInventoryFailedOutbox(ctx, tx, event, item)
        }
    }

    if err := s.repo.InsertInventoryDeductedOutbox(ctx, tx, event); err != nil {
        return kafka.Retryable(err)
    }

    return tx.Commit(ctx)
}
```

这里展示的是结构，实际代码要处理 commit 错误。

---

## 九、InsertProcessedEvent

```sql
INSERT INTO processed_events(event_id, topic, partition_id, offset_value, handler)
VALUES ($1, $2, $3, $4, $5)
ON CONFLICT (event_id) DO NOTHING;
```

判断影响行数：

```text
1 行：第一次处理。
0 行：重复消息。
```

---

## 十、DeductInventory

```sql
UPDATE inventory
SET available = available - $1,
    updated_at = now()
WHERE sku_id = $2
  AND available >= $1;
```

影响行数：

```text
1 行：扣减成功。
0 行：库存不足或商品不存在。
```

---

## 十一、重复消费测试

手动向 `order.created` 发送两条相同 event_id 的消息。

预期：

- 第一条扣库存。
- 第二条识别为重复。
- 库存不会再次减少。
- offset 仍然提交。

---

## 十二、本节练习

1. 实现 `OrderCreatedHandler`。
2. 实现 `InsertProcessedEvent`。
3. 实现 `DeductInventory`。
4. 手动发送重复消息。
5. 验证库存只扣一次。

---

## 十三、Repository 接口设计

为了让 handler 不直接拼 SQL，建议把数据库操作收敛到 repository：

```go
type Repository interface {
    InsertProcessedEvent(ctx context.Context, tx pgx.Tx, eventID string, msg kafka.ConsumedMessage) (bool, error)
    DeductInventory(ctx context.Context, tx pgx.Tx, skuID string, quantity int) (bool, error)
    InsertInventoryDeductedOutbox(ctx context.Context, tx pgx.Tx, event OrderCreatedEvent) error
    InsertInventoryFailedOutbox(ctx context.Context, tx pgx.Tx, event OrderCreatedEvent, item OrderItem) error
}
```

这里返回 `bool` 的方法都表达一个业务判断：

```text
InsertProcessedEvent:
  true  -> 第一次处理
  false -> 重复事件

DeductInventory:
  true  -> 扣减成功
  false -> 库存不足或 SKU 不存在
```

不要把这种判断隐藏在错误里。库存不足不是数据库错误，它是业务结果。

---

## 十四、InsertProcessedEvent 实现

示例实现：

```go
func (r *PGRepository) InsertProcessedEvent(
    ctx context.Context,
    tx pgx.Tx,
    eventID string,
    msg kafka.ConsumedMessage,
) (bool, error) {
    tag, err := tx.Exec(ctx, `
        INSERT INTO processed_events(event_id, topic, partition_id, offset_value, handler)
        VALUES ($1, $2, $3, $4, $5)
        ON CONFLICT (event_id) DO NOTHING
    `, eventID, msg.Topic, msg.Partition, msg.Offset, "inventory-service")
    if err != nil {
        return false, err
    }

    return tag.RowsAffected() == 1, nil
}
```

这个方法不要在内部 `commit`。事务边界应该由 service 控制，因为扣库存和写结果事件必须在同一个事务里。

---

## 十五、DeductInventory 实现

```go
func (r *PGRepository) DeductInventory(
    ctx context.Context,
    tx pgx.Tx,
    skuID string,
    quantity int,
) (bool, error) {
    tag, err := tx.Exec(ctx, `
        UPDATE inventory
        SET available = available - $1,
            updated_at = now()
        WHERE sku_id = $2
          AND available >= $1
    `, quantity, skuID)
    if err != nil {
        return false, err
    }

    return tag.RowsAffected() == 1, nil
}
```

这里把 `available >= quantity` 放进 SQL，而不是先查再更新。

不要写成：

```text
SELECT available
if available >= quantity:
  UPDATE inventory
```

因为并发下两条消息可能同时读到库存足够，然后都扣减成功。把条件放进 `UPDATE`，数据库会用行锁保证判断和扣减是一个原子动作。

---

## 十六、库存不足怎么处理

库存不足不应该进入 DLQ。

原因是 DLQ 表示“消息坏了或程序无法处理”，而库存不足是正常业务分支。

更合理的做法是发布一个业务事件：

```text
inventory.deduct_failed
```

事件示例：

```json
{
  "event_id": "evt_inventory_failed_order_1001",
  "event_type": "inventory.deduct_failed",
  "version": 1,
  "occurred_at": "2026-07-05T10:20:00Z",
  "data": {
    "order_id": "order_1001",
    "sku_id": "sku_1",
    "reason": "INSUFFICIENT_STOCK"
  }
}
```

学习项目里可以先把这个事件写入 outbox，后续再由 worker 发布。

---

## 十七、什么时候提交 Kafka offset

处理成功以后才提交 offset：

```text
数据库事务 commit 成功
-> handler 返回 nil
-> consumer commit offset
```

重复消息也要提交 offset：

```text
processed_events 冲突
-> 说明之前已经处理成功
-> commit 数据库事务
-> commit kafka offset
```

数据库错误不要提交 offset，除非你已经把原消息成功写入 retry topic。否则消息可能丢失。

---

## 十八、手动验证重复消费

先查库存：

```sql
SELECT sku_id, available
FROM inventory
WHERE sku_id = 'sku_1';
```

发送两条相同 `event_id` 的消息：

```bash
kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic order.created \
  --property parse.key=true \
  --property key.separator=:
```

输入两次：

```text
order_1001:{"event_id":"evt_order_created_order_1001","event_type":"order.created","version":1,"data":{"order_id":"order_1001","user_id":"user_88","items":[{"sku_id":"sku_1","quantity":2}]}}
```

再次查库存：

```sql
SELECT sku_id, available
FROM inventory
WHERE sku_id = 'sku_1';
```

预期只减少 `2`，不是减少 `4`。

---

## 十九、常见错误

### 1. 先提交 offset 再提交数据库事务

如果 offset 已提交，但数据库事务失败，这条消息不会再被消费，库存也不会扣减。

### 2. 用 order_id 替代 event_id 做幂等

一个订单可能产生多个事件，比如 `order.created`、`order.cancelled`。幂等键应该是事件级别的 `event_id`。

### 3. 把库存不足当成可重试错误

库存不足重试一百次也大概率不会成功，应该进入业务失败流程，而不是 retry topic。

### 4. handler 里吞掉错误

如果数据库超时后返回 `nil`，consumer 会提交 offset，消息就真的丢了。错误必须向上返回给 consumer 框架分类处理。

---

## 二十、面试表达

可以这样解释这段设计：

```text
库存服务消费 order.created 时，我不会只依赖 Kafka offset 做可靠性。
每条业务事件都有 event_id，consumer 在本地事务里先插入 processed_events。
如果插入成功，说明第一次处理，然后执行带条件的库存扣减；如果唯一键冲突，说明之前已经处理过，直接返回成功并提交 offset。
这样即使 Kafka 重复投递、consumer 重启或者 offset 提交失败，也不会重复扣库存。
```
