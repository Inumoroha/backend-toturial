# 08. 订单 API 与 Outbox 落地

## 1. 本节目标

实现订单 API 时，必须使用 Transactional Outbox。

也就是说：

```text
写 orders
写 outbox_messages
同一个数据库事务提交
```

不要在 API 中直接发布 RabbitMQ。

## 2. 创建订单事务

流程：

```text
开启事务
插入 orders
插入 order_items
插入 outbox order.created
插入 outbox order.timeout.check
提交事务
```

伪代码：

```go
func CreateOrder(ctx context.Context, req CreateOrderRequest) (int64, error) {
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return 0, err
	}
	defer tx.Rollback()

	orderID, err := insertOrder(ctx, tx, req)
	if err != nil {
		return 0, err
	}

	if err := insertOrderItems(ctx, tx, orderID, req.Items); err != nil {
		return 0, err
	}

	if err := insertOutbox(ctx, tx, NewOrderCreatedEvent(orderID, req)); err != nil {
		return 0, err
	}

	if err := insertOutbox(ctx, tx, NewOrderTimeoutCheckTask(orderID)); err != nil {
		return 0, err
	}

	return orderID, tx.Commit()
}
```

## 3. 支付订单事务

流程：

```text
开启事务
条件更新 orders status=paid
插入 outbox order.paid
提交事务
```

条件更新：

```sql
UPDATE orders
SET status = 'paid'
WHERE id = ?
  AND status IN ('pending_inventory', 'pending_payment');
```

如果影响行数为 0，要查询订单当前状态。

不要重复发布 `order.paid`。

## 4. Outbox 插入函数

Outbox 数据：

```go
type OutboxMessage struct {
	MessageID     string
	AggregateType string
	AggregateID   string
	EventType     string
	ExchangeName  string
	RoutingKey    string
	Payload       []byte
	Headers       []byte
}
```

插入：

```sql
INSERT INTO outbox_messages (
    message_id,
    aggregate_type,
    aggregate_id,
    event_type,
    exchange_name,
    routing_key,
    payload,
    headers,
    status
) VALUES (?, ?, ?, ?, ?, ?, ?, ?, 'pending');
```

## 5. API 返回语义

创建订单 API 返回：

```json
{
  "order_id": 1,
  "status": "pending_inventory"
}
```

这不代表：

```text
库存已经预占成功。
```

异步系统要让状态表达中间过程。

## 6. 为什么 API 不直接发布消息

错误方式：

```text
提交订单事务
直接 publish RabbitMQ
```

问题：

```text
数据库成功但 publish 失败，事件丢失。
```

Outbox 方式：

```text
数据库成功时，待发布消息一定也在数据库里。
```

## 7. 本节小结

订单 API 的关键不是 HTTP 路由，而是：

```text
业务表和 outbox 表同事务。
```

下一节实现 Outbox Worker 和可靠发布。

