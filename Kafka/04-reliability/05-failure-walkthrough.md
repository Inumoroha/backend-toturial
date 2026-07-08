# 05 失败链路逐步推演

本节目标：把“消息丢失、重复消费、retry、DLQ、outbox”放进一条真实业务链路里推演。只理解概念不够，必须知道每一步失败时系统会怎样恢复。

## 场景：创建订单后扣减库存

链路：

```text
Client -> order-service -> database orders + outbox -> outbox-worker -> Kafka order.created -> inventory-service -> database inventory
```

## 情况 1：订单写库成功，Kafka 发送失败

### 如果没有 Outbox

流程：

```text
写 orders 成功
发送 order.created 失败
HTTP 返回成功或失败都很尴尬
```

后果：

- 如果返回成功，下游永远不知道订单创建。
- 如果返回失败，用户可能重试，但数据库里已经有订单。
- 需要人工补偿。

### 使用 Outbox

流程：

```text
同一个事务写 orders 和 outbox_events
事务提交成功
outbox-worker 发送 Kafka
发送失败就更新 retry_count 或下次继续扫描
```

后果：

- 订单和待发送事件不会分离。
- Kafka 临时失败可以重试。
- 事件最终能发出去。

## 情况 2：Outbox 发送 Kafka 成功，但标记 sent 失败

流程：

```text
outbox-worker 发送 order.created 成功
更新 outbox_events.status=sent 失败
下一轮 worker 又扫描到同一条 outbox
再次发送 order.created
```

后果：

- Kafka 里可能出现重复事件。

处理：

- consumer 必须用 `event_id` 幂等。
- outbox event id 必须稳定，不能每次重试生成新 id。

结论：

```text
Outbox 解决事件丢失，不消灭重复消息
```

## 情况 3：Consumer 扣库存成功，提交 offset 失败

流程：

```text
inventory-service 收到 order.created
扣减库存成功
提交 offset 失败
consumer 重启
再次收到同一条 order.created
```

如果没有幂等：

- 库存重复扣减。

如果有幂等：

```text
检查 processed_events 发现 event_id 已处理
直接返回成功
再次提交 offset
```

结论：

```text
提交 offset 失败不是灾难，前提是业务处理幂等
```

## 情况 4：Consumer 业务失败，不提交 offset

适合非常短暂的错误：

- 数据库瞬时超时。
- 网络抖动。

但如果错误长期存在：

- 当前 partition 会被同一条消息卡住。
- 后续正常消息也无法处理。
- lag 持续增长。

处理策略：

1. 本地短重试 2 到 3 次。
2. 仍失败则写 retry topic。
3. 写 retry 成功后提交原 offset。

## 情况 5：消息格式错误

例如：

```text
bad-json
```

这类错误重试没有意义。

正确处理：

```text
解析失败
构造 DLQ 消息
写入 order.created.dlq
写 DLQ 成功
提交原 offset
告警或记录
```

错误处理不能只是打印日志，否则坏消息会在主链路消失。

## 情况 6：Retry Topic 再次失败

设计最大重试次数：

```text
order.created -> retry.1m -> retry.5m -> retry.30m -> dlq
```

每次 retry 消息要带：

- `event_id`
- `attempt`
- `original_topic`
- `original_partition`
- `original_offset`
- `last_error`
- `first_failed_at`
- `last_failed_at`

超过最大次数进入 DLQ。

## 正确提交 Offset 的决策表

| 处理结果 | 下一步 | 是否提交原 offset |
| --- | --- | --- |
| 业务成功 | 无 | 是 |
| 重复消息，已处理过 | 无 | 是 |
| 可重试错误，本地重试未耗尽 | 原地重试 | 否 |
| 可重试错误，已写 retry topic | 等 retry consumer | 是 |
| 不可重试错误，已写 DLQ | 等人工处理 | 是 |
| 写 retry 失败 | 保留原消息 | 否 |
| 写 DLQ 失败 | 保留原消息 | 否 |
| 进程退出，当前消息未完成 | 重启后重读 | 否 |

## Go 后端事务边界

库存扣减推荐：

```text
begin db tx
insert processed_events(event_id)
if duplicate:
    commit tx
    return success
check inventory
deduct inventory
insert inventory_logs
commit tx
commit kafka offset
```

重点：

- `processed_events` 和库存扣减必须在同一个数据库事务中。
- Kafka offset 提交不在数据库事务中，所以仍然可能重复消费。
- 重复消费靠 `processed_events` 兜住。

## 本节验收

你需要能完整回答：

1. Outbox 为什么仍然需要 consumer 幂等？
2. 扣库存成功但 offset 提交失败会怎样？
3. 写 DLQ 失败为什么不能提交 offset？
4. 坏消息为什么应该进入 DLQ，而不是无限重试？
5. retry topic 中为什么要保留原 topic、partition、offset？

