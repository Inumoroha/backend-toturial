# 09. Saga 与最终一致性入门

## 1. 为什么需要 Saga

一个订单流程可能跨多个服务：

```text
order-service
inventory-service
payment-service
coupon-service
notification-service
```

单个数据库事务无法覆盖所有服务。

如果你强行用同步调用串起来：

```text
创建订单 -> 扣库存 -> 扣优惠券 -> 支付 -> 发通知
```

任何一步失败，都可能让系统处在中间状态。

Saga 用来管理这种跨服务长事务。

## 2. Saga 的基本思想

Saga 把一个大事务拆成多个本地事务。

每一步成功后，触发下一步。

如果某一步失败，执行补偿动作。

例如：

```text
创建订单
  -> 预占库存
  -> 锁定优惠券
  -> 等待支付
```

如果库存预占失败：

```text
取消订单
```

如果支付超时：

```text
取消订单
释放库存
释放优惠券
```

## 3. 编排式 Saga

编排式 Saga 有一个协调者。

```text
order-saga-orchestrator
  -> 命令库存服务预占
  -> 命令优惠券服务锁定
  -> 命令订单服务确认
  -> 失败时命令各服务补偿
```

优点：

- 流程集中，容易看懂。
- 复杂流程更可控。

缺点：

- 协调者容易变重。
- 服务之间耦合更明显。

## 4. 协同式 Saga

协同式 Saga 通过事件驱动。

```text
order-service 发布 order.created
inventory-service 消费并发布 inventory.reserved
coupon-service 消费并发布 coupon.locked
order-service 根据事件更新状态
```

优点：

- 服务解耦。
- 事件流自然扩展。

缺点：

- 流程分散。
- 排查更难。
- 需要更好的日志、状态机和监控。

## 5. RabbitMQ 更适合做什么

RabbitMQ 可以承载 Saga 中的事件和命令：

```text
order.created
inventory.reserved
inventory.reserve_failed
payment.succeeded
payment.failed
order.cancelled
```

但 RabbitMQ 不会自动帮你管理 Saga 状态。

你仍然需要：

- Saga 状态表。
- 业务状态机。
- 幂等消费者。
- 超时补偿。
- 失败告警。

## 6. 简化订单 Saga

状态：

```text
pending_inventory
pending_payment
paid
cancelled
```

流程：

```text
order.created
  -> inventory-service 预占库存
  -> inventory.reserved
  -> order-service 更新为 pending_payment
  -> 用户支付
  -> payment.succeeded
  -> order-service 更新为 paid
```

失败：

```text
inventory.reserve_failed
  -> order-service 更新为 cancelled
  -> 发布 order.cancelled
```

支付超时：

```text
order.timeout.check
  -> order-service 条件取消
  -> 发布 order.cancelled
  -> inventory-service 释放库存
```

## 7. Saga 状态表

可以设计：

```sql
CREATE TABLE order_sagas (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT NOT NULL,
    status VARCHAR(32) NOT NULL,
    current_step VARCHAR(64) NOT NULL,
    last_event VARCHAR(64) NULL,
    last_error TEXT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_order_id (order_id)
);
```

状态示例：

```text
started
inventory_reserved
payment_pending
completed
compensating
cancelled
failed
```

## 8. 最终一致性如何向用户展示

异步系统中，不是每个动作立即完成。

用户创建订单后可以看到：

```text
订单已创建，正在确认库存
```

库存确认后：

```text
待支付
```

支付后：

```text
已支付
```

失败后：

```text
订单已取消，原因：库存不足
```

不要让用户误以为所有步骤已经同步完成。

## 9. Saga 的坑

### 坑一：没有补偿动作

每个可失败步骤都要考虑补偿。

例如：

```text
预占库存 -> 释放库存
锁定优惠券 -> 释放优惠券
```

### 坑二：没有幂等

Saga 中每个事件都可能重复。

每个参与服务都要幂等。

### 坑三：没有超时

如果等待某个事件永远不来，Saga 会卡住。

要设计：

```text
timeout check
```

### 坑四：没有状态机

状态机是 Saga 的骨架。

没有状态机，重复和乱序事件会让业务混乱。

## 10. 本节小结

Saga 解决跨服务长事务问题。

你要记住：

- Saga = 多个本地事务 + 事件/命令 + 补偿。
- RabbitMQ 可以传递事件，但不管理业务状态。
- 必须有状态机。
- 必须有幂等。
- 必须有超时和补偿。
- 用户界面要接受最终一致性。

下一节学习 RabbitMQ 在 Go 项目中的封装建议。

