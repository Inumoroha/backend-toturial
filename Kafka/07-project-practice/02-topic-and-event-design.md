# 02 Topic 与事件设计

本节目标：为项目设计 topic、message key 和事件结构。

## Topic 列表

主业务 topic：

```text
order.created
order.cancelled
inventory.deducted
inventory.failed
payment.succeeded
payment.failed
notification.sent
```

重试和死信 topic：

```text
order.created.retry.1m
order.created.retry.5m
order.created.dlq

inventory.deducted.retry.1m
inventory.deducted.dlq
```

## Message Key

| Topic | Key | 原因 |
| --- | --- | --- |
| `order.created` | `order_id` | 同一订单事件保持顺序 |
| `inventory.deducted` | `order_id` | 和订单流程对齐 |
| `payment.succeeded` | `order_id` | 同一订单支付事件顺序 |
| `notification.sent` | `notification_id` | 通知自身唯一 |

## 订单创建事件

```json
{
  "event_id": "evt_order_created_1001",
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

## 库存扣减事件

```json
{
  "event_id": "evt_inventory_deducted_1001",
  "event_type": "inventory.deducted",
  "version": 1,
  "occurred_at": "2026-07-05T12:00:02Z",
  "data": {
    "order_id": "order_1001",
    "deductions": [
      {
        "sku_id": "sku_1",
        "quantity": 2
      }
    ]
  }
}
```

## 支付成功事件

```json
{
  "event_id": "evt_payment_succeeded_1001",
  "event_type": "payment.succeeded",
  "version": 1,
  "occurred_at": "2026-07-05T12:01:00Z",
  "data": {
    "order_id": "order_1001",
    "payment_id": "pay_9001",
    "amount": 19800
  }
}
```

## 事件设计规则

- 每个事件必须有 `event_id`。
- 每个事件必须有 `version`。
- 金额用整数分，不用浮点数。
- 时间使用 ISO 8601。
- 不在事件中放敏感信息。
- 不要让 consumer 依赖 producer 数据库里的内部字段。

## 本节练习

1. 为 `order.cancelled` 设计事件结构。
2. 为 `payment.failed` 设计事件结构。
3. 为每个 topic 选择 partition 数量。
4. 思考：如果一个订单包含多个商品，库存扣减 key 应该用 `order_id` 还是 `sku_id`？

