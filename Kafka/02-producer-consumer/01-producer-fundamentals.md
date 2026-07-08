# 01 Producer 基础

Producer 的职责是把业务事件可靠、高效地写入 Kafka。对 Go 后端来说，producer 通常藏在业务服务内部，例如订单服务创建订单后发布 `order.created`。

## 一条消息包含什么

Kafka 消息常见字段：

- topic：写入哪个 topic。
- key：决定分区，也用于业务定位。
- value：消息正文。
- headers：元数据，例如 trace id、event id、schema version。
- timestamp：消息时间。

Go 后端推荐事件结构：

```json
{
  "event_id": "evt_20260705_000001",
  "event_type": "order.created",
  "version": 1,
  "occurred_at": "2026-07-05T12:00:00Z",
  "data": {
    "order_id": "order_1001",
    "user_id": "user_88",
    "amount": 19900
  }
}
```

## Message Key 如何选择

key 会影响 partition 分配。

常见选择：

| 场景 | 推荐 key |
| --- | --- |
| 订单事件 | `order_id` |
| 用户事件 | `user_id` |
| 商品库存事件 | `sku_id` |
| 支付事件 | `payment_id` 或 `order_id` |

选择 key 的原则：

- 同一业务实体需要顺序时，用这个实体的 ID。
- 不要用随机 UUID 当 key，否则顺序性失去意义。
- 避免热点 key，例如所有消息都用同一个 key。

## Producer 发送流程

简化流程：

1. 业务代码构造消息。
2. producer 根据 topic 和 key 选择 partition。
3. 消息进入客户端缓冲区。
4. 客户端按 batch 发送给 broker。
5. broker 返回确认。
6. producer 得到成功或失败结果。

## 可靠性参数

### acks

`acks` 决定 producer 等待 broker 怎样确认。

| 配置 | 含义 | 风险 |
| --- | --- | --- |
| `acks=0` | 不等待确认 | 可能大量丢消息 |
| `acks=1` | leader 写入后确认 | leader 崩溃时可能丢 |
| `acks=all` | ISR 副本确认后返回 | 可靠性更高，延迟更高 |

生产中关键业务通常使用 `acks=all`。

### retries

发送失败时是否重试。

需要注意：

- 重试可能导致重复消息。
- 如果没有幂等 producer，重试可能影响顺序。
- Go 业务层仍然要能处理重复事件。

### enable.idempotence

幂等 producer 可以减少 producer 重试导致的重复写入。

关键业务建议开启。

注意：producer 幂等不等于业务幂等。它只能约束 producer 到 broker 之间的一部分重复问题，consumer 业务仍然要做去重。

## 吞吐参数

### batch.size

批次越大，吞吐可能越高，但单条消息等待时间可能变长。

### linger.ms

producer 等待更多消息组成 batch 的时间。

- 追求吞吐：适当增大。
- 追求低延迟：保持较小。

### compression.type

常用：

- `snappy`
- `lz4`
- `zstd`

压缩可以减少网络和磁盘压力，但会增加 CPU 开销。

## Go 后端发送建议

- 封装统一 producer，不要在业务代码里到处 new producer。
- 消息必须带 `event_id`。
- 日志必须打印 topic、key、event_id。
- 发送失败要区分可重试和不可重试。
- 服务退出时要 flush 或 close producer。
- 关键业务优先保证可靠性，再考虑吞吐。

## 典型发送策略

普通业务事件：

```text
acks=all
enable.idempotence=true
compression.type=zstd 或 lz4
linger.ms=5~20
retries 合理开启
```

日志采集类事件：

```text
acks=1 或 all
compression.type=lz4
batch.size 较大
linger.ms 适当增大
```

极低延迟事件：

```text
linger.ms 较小
batch.size 不宜过大
acks 根据业务可靠性取舍
```

## 本节练习

1. 创建一个 3 partition 的 topic。
2. 分别用有 key 和无 key 的方式发送消息。
3. 观察消息分区分布。
4. 解释为什么同一个 key 通常会进入同一个 partition。
5. 思考：订单事件应该用 `order_id` 还是 `user_id` 做 key？

