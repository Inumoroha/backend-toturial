# 09. Change Streams 基础

Change Streams 可以监听 MongoDB 中的数据变更。它常用于订单状态变更通知、缓存更新、搜索索引同步、审计事件等场景。

## 1. Change Streams 是什么

Change Streams 允许应用订阅集合、数据库或整个部署中的变更事件。

它基于复制集的 oplog。

因此通常要求 MongoDB 运行在复制集或分片集群上。

## 2. 适合场景

适合：

- 订单状态变化通知。
- 商品变更后同步搜索索引。
- 用户资料变更后刷新缓存。
- 审计日志。
- 异步事件处理。

不适合：

- 替代事务。
- 替代消息队列的所有能力。
- 处理不可丢失但没有恢复机制的关键任务。

如果事件非常关键，建议结合 outbox 模式和可靠消息系统。

## 3. 准备数据

```javascript
use change_stage7
db.orders.insertOne({
  order_no: "O-CS-001",
  status: "pending_payment",
  total_amount: 9900,
  created_at: new Date()
})
```

## 4. mongosh 监听变更

打开一个终端：

```powershell
mongosh "mongodb://localhost:27017,localhost:27018,localhost:27019/?replicaSet=rs0"
```

执行：

```javascript
use change_stage7

const stream = db.orders.watch()

while (stream.hasNext()) {
  printjson(stream.next())
}
```

再打开另一个终端，更新订单：

```javascript
use change_stage7

db.orders.updateOne(
  { order_no: "O-CS-001" },
  {
    $set: {
      status: "paid",
      paid_at: new Date(),
      updated_at: new Date()
    }
  }
)
```

第一个终端应该能看到变更事件。

## 5. 只监听 update

```javascript
const stream = db.orders.watch([
  {
    $match: {
      operationType: "update"
    }
  }
])
```

## 6. fullDocument

默认 update 事件不一定包含完整更新后的文档。

可以：

```javascript
const stream = db.orders.watch(
  [],
  { fullDocument: "updateLookup" }
)
```

这样 update 时会查出完整更新后文档。

注意：这会增加额外读取成本。

## 7. Resume Token

Change Streams 事件中包含 resume token。

应用重启后可以从上次处理位置继续。

生产中必须保存处理进度，否则服务重启可能漏事件或重复处理。

事件处理也要幂等。

## 8. Change Streams 和消息队列

Change Streams 可以感知数据库变更，但它不是完整消息系统。

如果业务需要：

- 多消费者组。
- 复杂重试。
- 死信队列。
- 高吞吐事件流。
- 跨系统事件治理。

通常还需要 Kafka、RabbitMQ、Redis Stream 等消息系统。

## 9. 本节练习

完成：

1. 启动复制集。
2. 在 `orders` 集合打开 watch。
3. 插入订单，观察 insert 事件。
4. 更新订单状态，观察 update 事件。
5. 使用 `$match` 只监听 update。
6. 使用 `fullDocument: "updateLookup"`。
7. 写出 Change Streams 适合和不适合的场景。

## 10. 本节小结

你需要记住：

- Change Streams 用于监听数据变更。
- 它依赖复制集或分片集群。
- update 事件默认不一定有完整文档。
- 生产中要保存 resume token。
- 事件处理必须幂等。
- 它不完全替代消息队列。

## 11. Change Streams 适合什么

适合：

- 订单状态变更后更新搜索索引。
- 用户资料变更后刷新缓存。
- 审计日志采集。
- 同步部分数据到分析系统。
- 低频或中频事件驱动任务。

不适合：

- 替代高吞吐消息队列。
- 承担复杂业务编排。
- 处理无法幂等的外部副作用。
- 在没有 resume 机制时做关键事件投递。

如果事件绝对不能丢，通常要考虑业务写入时同时写 outbox 集合，再由可靠 worker 投递。

## 12. resume token 为什么重要

Change Streams 事件中包含 resume token。服务重启后，如果不保存 token，只能从当前时刻继续监听，重启期间的事件可能丢失。

处理流程：

```text
读取 resume token
从 token 后继续 watch
处理事件
业务处理成功
保存新的 resume token
```

注意保存顺序。通常要保证“业务处理成功后再推进 token”，否则可能跳过未处理事件。

## 13. Go 中保存 resume token 的思路

伪代码：

```go
for stream.Next(ctx) {
    var event bson.M
    if err := stream.Decode(&event); err != nil {
        return err
    }

    if err := handleEvent(ctx, event); err != nil {
        return err
    }

    token := stream.ResumeToken()
    if err := saveResumeToken(ctx, "orders-worker", token); err != nil {
        return err
    }
}
```

生产中 `handleEvent` 必须幂等。因为服务可能在处理成功后、保存 token 前崩溃，重启后会再次处理同一事件。

## 14. 幂等处理示例

假设监听订单支付成功后发送通知，可以记录已处理事件：

```javascript
db.processed_events.createIndex(
  { worker: 1, event_id: 1 },
  { unique: true }
)
```

处理时：

```text
先插入 processed_events
如果重复键，说明处理过，直接跳过
否则执行业务动作
保存 resume token
```

对于发送短信、调用外部 HTTP 这种不可回滚动作，要格外谨慎。最好把外部投递也做成带幂等键的任务。

## 15. fullDocument 的成本

update 事件默认可能只包含变更描述，不包含完整文档。使用：

```go
options.ChangeStream().SetFullDocument(options.UpdateLookup)
```

可以在 update 时查回完整文档，但这会增加额外读取成本。

适合：

- 事件处理需要完整文档。
- 更新频率可控。

不适合：

- 高频更新集合。
- 大文档集合。
- 只需要少数字段的场景。

## 16. 监听管道要尽量过滤

只监听订单状态更新：

```javascript
[
  {
    "$match": {
      "operationType": "update",
      "ns.coll": "orders",
      "updateDescription.updatedFields.status": { "$exists": true }
    }
  }
]
```

不要无脑监听整个数据库所有变更再在应用里过滤，这会浪费大量资源。

## 17. 生产检查清单

```text
是否保存 resume token？
token 保存是否在业务处理成功之后？
事件处理是否幂等？
是否处理重复事件？
是否处理 stream 中断重连？
是否限制监听范围？
是否监控处理延迟？
是否有死信或失败重试机制？
```

## 18. 官方文档延伸阅读

- [Change Streams](https://www.mongodb.com/docs/manual/changeStreams/)
- [Go Driver Change Streams](https://www.mongodb.com/docs/drivers/go/current/usage-examples/changestream/)
