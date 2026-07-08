# 05. writeConcern 写关注与 majority

writeConcern 决定写入需要被确认到什么程度才算成功。它是 MongoDB 一致性和高可用设计的核心概念之一。

## 1. w:1

`w:1` 表示 primary 确认写入就返回成功。

优点：

- 延迟较低。
- 吞吐更高。

风险：

- 如果 primary 写入后尚未复制到多数节点就故障，极端情况下可能回滚。

## 2. majority

`w: "majority"` 表示写入需要被多数有投票权节点确认。

三节点复制集：

```text
majority = 2
```

写入要至少被 2 个节点确认。

优点：

- 更强持久性。
- 更能抵抗 primary 故障导致的回滚。

代价：

- 写入延迟增加。
- 依赖多数节点可用。

## 3. mongosh 中测试 majority

```javascript
db.orders.insertOne(
  {
    order_no: "O-MAJORITY-001",
    status: "paid",
    created_at: new Date()
  },
  {
    writeConcern: { w: "majority", wtimeout: 5000 }
  }
)
```

`wtimeout` 表示等待写关注确认的超时时间。

如果多数节点不可用，可能超时。

## 4. 什么时候用 majority

建议考虑：

- 订单创建。
- 支付流水。
- 钱包交易。
- 库存扣减。
- 用户注册。
- 幂等关键记录。

不一定所有写都要 majority：

- 浏览量。
- 临时日志。
- 非关键埋点。
- 可丢失缓存数据。

## 5. writeConcern 和事务

事务通常会结合 writeConcern。

Go 中：

```go
txnOpts := options.Transaction().
    SetWriteConcern(writeconcern.Majority())
```

这样事务提交需要 majority 确认。

## 6. writeConcern 不是业务成功的全部

`majority` 只能说明写入被多数节点确认。

它不能替代：

- 唯一索引。
- 幂等设计。
- 状态条件。
- 业务校验。

例如重复下单仍然要用 `idempotency_key`。

## 7. wtimeout

```javascript
writeConcern: { w: "majority", wtimeout: 5000 }
```

如果 5 秒内没有达到 majority，会返回错误。

注意：写关注超时并不一定表示写入完全失败。应用需要根据幂等键查询最终状态。

这也是幂等设计重要的原因。

## 8. Go 中设置 writeConcern

Client 级别：

```go
clientOpts := options.Client().
    ApplyURI(uri).
    SetWriteConcern(writeconcern.Majority())
```

Collection 级别：

```go
coll := db.Collection(
    "orders",
    options.Collection().SetWriteConcern(writeconcern.Majority()),
)
```

事务级别：

```go
txnOpts := options.Transaction().
    SetWriteConcern(writeconcern.Majority())
```

## 9. 本节练习

完成：

1. 用默认 writeConcern 插入一条订单。
2. 用 majority 插入一条订单。
3. 停掉一个 secondary，再用 majority 写入。
4. 停掉两个节点，再观察 majority 写入是否成功。
5. 写出 w:1 和 majority 的差异。

## 10. 本节小结

你需要记住：

- `w:1` 是 primary 确认。
- `majority` 是多数节点确认。
- 重要写入建议考虑 majority。
- majority 会增加延迟和可用性要求。
- 写关注超时后要结合幂等键确认最终状态。

## 11. w:1 和 majority 的真实差异

`w:1` 表示 primary 写入成功就向客户端确认。  
`majority` 表示写入被多数投票节点确认后再向客户端确认。

在三节点复制集中：

```text
w:1
  primary 写成功 -> 返回成功

majority
  primary 写成功
  至少一个 secondary 也确认
  -> 返回成功
```

`majority` 的好处是降低 primary 刚写完就宕机导致数据回滚的风险。代价是写入延迟更高，并且当多数节点不可用时写入可能失败。

## 12. writeConcern 超时不等于一定失败

一个常见坑：

```text
客户端收到 writeConcern timeout
```

这并不总表示写入没有发生。它可能表示：

- primary 已写入。
- 但等待多数派确认超时。
- 后续该写入可能继续复制成功。

所以业务不能简单地收到超时就再创建一笔新订单。必须依赖幂等键或业务唯一索引确认最终状态。

Go 中建议：

```go
_, err := orders.InsertOne(ctx, order)
if err != nil {
    existing, findErr := findByRequestID(ctx, tenantID, requestID)
    if findErr == nil {
        return existing, nil
    }
    return nil, err
}
```

这个模式不是所有错误都直接重试，而是先按幂等键查最终状态。

## 13. Go 中设置 writeConcern

客户端级别：

```go
client, err := mongo.Connect(options.Client().
    ApplyURI(uri).
    SetWriteConcern(writeconcern.Majority()))
```

集合级别：

```go
orders := db.Collection("orders", options.Collection().
    SetWriteConcern(writeconcern.Majority()))
```

单次操作也可以通过 option 设置，但实际项目中更常按集合或业务仓储统一配置。

## 14. 不同业务的选择

| 业务 | 推荐 writeConcern | 说明 |
| --- | --- | --- |
| 钱包流水 | majority | 金额数据，不能轻易丢 |
| 支付回调 | majority | 必须结合幂等键 |
| 订单创建 | majority | 重要业务数据 |
| 浏览量 +1 | w:1 或异步 | 可补偿、可最终一致 |
| 埋点日志 | w:1 / 批量 / 队列 | 取决于可丢失程度 |

重要业务还要配合：

- 唯一索引。
- 幂等键。
- 事务。
- 业务状态机。

writeConcern 不是单独的安全魔法。

## 15. majority 的可用性代价

如果三节点复制集中两个节点不可用：

```text
只剩 primary 一个节点
```

`w:1` 也许还能写，但 `majority` 无法满足多数派确认。  
这就是一致性和可用性的取舍。

你需要根据业务选择：

```text
宁可短暂不可写，也不接受丢关键数据？
还是宁可继续写，后续再补偿？
```

金融、订单、库存通常偏向前者。日志、埋点、非核心统计可能偏向后者。

## 16. 写关注评审模板

```md
# writeConcern 评审

## 集合

## 业务风险

## 推荐 writeConcern

## 是否需要幂等键

## 是否需要唯一索引

## 超时后如何确认最终状态

## 是否允许补偿
```

## 17. 官方文档延伸阅读

- [Write Concern](https://www.mongodb.com/docs/manual/reference/write-concern/)
- [Causal Consistency and Read/Write Concerns](https://www.mongodb.com/docs/manual/core/causal-consistency-read-write-concerns/)
