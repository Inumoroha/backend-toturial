# 05. Transactional Outbox：事务性发件箱

## 1. 它解决什么问题

业务中最经典的问题：

```text
数据库事务成功了，但消息发布失败了。
```

例如订单创建：

```text
写 orders 表成功
发布 order.created 失败
```

结果：

```text
订单存在
库存服务不知道
通知服务不知道
审计服务不知道
```

这就是数据库和消息系统之间的一致性问题。

## 2. 错误做法：事务后直接发布

伪代码：

```go
tx.Begin()
insertOrder(tx)
tx.Commit()

publisher.Publish("order.created")
```

问题：

```text
tx.Commit 成功后，Publish 可能失败。
```

如果反过来：

```go
publisher.Publish("order.created")
tx.Commit()
```

又可能：

```text
消息发出去了，但数据库事务失败。
```

这两种都不可靠。

## 3. Outbox 的核心思想

Transactional Outbox 的核心是：

```text
在同一个数据库事务里，同时写业务表和 outbox 表。
```

然后由后台 worker 扫描 outbox 表，异步发布消息到 RabbitMQ。

流程：

```text
业务请求
  -> 开启数据库事务
  -> 写 orders 表
  -> 写 outbox_messages 表
  -> 提交事务

Outbox Worker
  -> 扫描未发送 outbox
  -> 发布 RabbitMQ
  -> publisher confirm 成功
  -> 标记 outbox 已发送
```

这样可以保证：

```text
只要订单写入成功，对应的待发布消息也一定写入成功。
```

## 4. Outbox 表设计

```sql
CREATE TABLE outbox_messages (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    message_id VARCHAR(128) NOT NULL,
    aggregate_type VARCHAR(64) NOT NULL,
    aggregate_id VARCHAR(64) NOT NULL,
    event_type VARCHAR(64) NOT NULL,
    routing_key VARCHAR(128) NOT NULL,
    exchange_name VARCHAR(128) NOT NULL,
    payload JSON NOT NULL,
    headers JSON NULL,
    status VARCHAR(32) NOT NULL,
    retry_count INT NOT NULL DEFAULT 0,
    last_error TEXT NULL,
    next_retry_at TIMESTAMP NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    sent_at TIMESTAMP NULL,
    UNIQUE KEY uk_message_id (message_id),
    KEY idx_status_next_retry (status, next_retry_at)
);
```

状态建议：

```text
pending
publishing
sent
failed
dead
```

## 5. 创建订单时写 Outbox

事务内：

```sql
INSERT INTO orders (...);

INSERT INTO outbox_messages (
    message_id,
    aggregate_type,
    aggregate_id,
    event_type,
    routing_key,
    exchange_name,
    payload,
    status
) VALUES (
    'msg-order-created-1001',
    'order',
    '1001',
    'order.created',
    'order.created',
    'order.events',
    '{...}',
    'pending'
);
```

关键点：

```text
orders 和 outbox_messages 在同一个本地事务中提交。
```

## 6. Go 伪代码

```go
func CreateOrder(ctx context.Context, req CreateOrderRequest) error {
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return err
	}
	defer tx.Rollback()

	orderID, err := insertOrder(ctx, tx, req)
	if err != nil {
		return err
	}

	event := NewOrderCreatedEvent(orderID, req.UserID, req.Items)
	if err := insertOutboxMessage(ctx, tx, event); err != nil {
		return err
	}

	return tx.Commit()
}
```

这里没有直接发布 RabbitMQ。

发布交给 Outbox Worker。

## 7. Outbox Worker 为什么可靠

如果 RabbitMQ 临时不可用：

```text
outbox 记录仍在数据库里
worker 稍后重试发布
```

如果 worker 发布成功但标记 sent 前崩溃：

```text
outbox 仍是 pending/publishing
worker 可能再次发布
```

这会导致重复消息，所以消费者必须幂等。

Outbox 解决的是：

```text
业务数据和“待发布消息”的一致性。
```

它不保证：

```text
消费者只处理一次。
```

## 8. Outbox 的代价

Outbox 会带来：

- 多一张表。
- 多一个 worker。
- 消息发布有轻微延迟。
- outbox 表需要清理归档。
- 消费者仍要幂等。

但对于关键业务，这是非常值得的。

## 9. 什么时候需要 Outbox

强烈建议使用：

- 订单创建事件。
- 支付成功事件。
- 资金流水事件。
- 库存变更事件。
- 优惠券发放事件。
- 任何“数据库成功但消息不能丢”的场景。

可以不使用：

- 非关键日志。
- 可丢弃指标。
- 临时调试消息。
- 失败影响很小的通知类任务。

## 10. 本节小结

Transactional Outbox 的核心：

```text
业务表 + outbox 表在同一个数据库事务中写入。
```

它解决：

```text
数据库事务成功但消息发布失败。
```

但它仍然要求：

- Outbox Worker 可靠发布。
- 消费者幂等。
- outbox 表监控和清理。

下一节学习 Outbox Worker 的 Go 实现思路。

