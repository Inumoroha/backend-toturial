# 02 幂等 Consumer 设计

本节目标：学会让 consumer 可以安全处理重复消息。

## 什么是幂等

幂等是指同一个操作执行一次和执行多次，最终结果一致。

例如：

```text
set order status = PAID
```

执行多次结果仍然是 `PAID`。

但下面这个操作不是幂等：

```text
user balance = user balance + 100
```

重复执行会多加钱。

## 为什么 consumer 必须幂等

Kafka consumer 重复消费是正常现象，不是异常现象。

只要使用至少一次语义，就必须接受：

- 同一条消息可能处理多次。
- 同一业务事件可能被重新投递。
- 人工修复 DLQ 时可能重放历史消息。

## 幂等键

每条业务事件必须有稳定唯一的 `event_id`。

例如：

```json
{
  "event_id": "evt_order_created_1001",
  "event_type": "order.created",
  "data": {
    "order_id": "order_1001"
  }
}
```

不要每次重试都生成新的 event id。否则 consumer 无法识别它是同一个事件。

## 去重表

常见做法是在数据库中创建消息处理记录表。

示例：

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

处理流程：

1. 开启数据库事务。
2. 尝试插入 `event_id`。
3. 如果插入冲突，说明处理过，直接返回成功。
4. 执行业务变更。
5. 提交数据库事务。
6. 提交 Kafka offset。

## 业务唯一约束

有些业务可以直接用唯一约束幂等。

例如优惠券发放：

```sql
CREATE UNIQUE INDEX uk_coupon_order
ON user_coupons(user_id, coupon_template_id, source_order_id);
```

同一个订单重复触发优惠券发放时，第二次插入会因唯一索引失败，然后业务判断为已处理。

## 状态机幂等

订单状态流转适合状态机。

例如：

```text
CREATED -> PAID -> SHIPPED -> FINISHED
```

如果当前订单已经是 `PAID`，再次收到 `payment.succeeded` 可以直接返回成功。

但如果当前状态是 `CANCELLED`，收到 `payment.succeeded` 就需要进入异常处理。

## 外部接口幂等

调用外部系统时，也要传幂等键。

例如：

- 支付退款：`refund_id`
- 发券：`grant_id`
- 短信：`message_id`

不要假设外部接口不会被重复调用。

## 本节练习

1. 为 `order.created` 设计 `event_id` 生成规则。
2. 写出去重表结构。
3. 思考：库存扣减应该用去重表、唯一索引还是状态机？
4. 思考：如果业务成功但插入去重表失败，会发生什么？

