# 07. readConcern、writeConcern、readPreference

这三个概念控制读写一致性和可用性取舍。第 6 阶段先建立基础认知，第 7 阶段学习复制集时会更深入。

## 1. writeConcern

writeConcern 决定写入需要被确认到什么程度才算成功。

常见：

```text
w: 1
w: "majority"
```

`w: 1` 表示 primary 确认写入即可。

`majority` 表示多数有投票权节点确认写入。

重要业务写入通常更偏向 `majority`，例如：

- 订单。
- 支付。
- 钱包。
- 库存。

## 2. readConcern

readConcern 决定读取什么一致性级别的数据。

事务里常见：

```text
snapshot
```

它让事务在一致性快照上读取数据。

学习阶段先记住：

- readConcern 控制读到什么级别的数据。
- 事务通常配合 snapshot。
- 不同级别有不同一致性和性能取舍。

## 3. readPreference

readPreference 决定从哪个节点读。

常见：

```text
primary
primaryPreferred
secondary
secondaryPreferred
nearest
```

默认通常从 primary 读。

从 secondary 读可能减轻 primary 压力，但可能读到稍旧数据。

重要强一致读，比如支付后立即查订单状态，通常应从 primary 读。

## 4. 事务中的读偏好

事务通常应使用 primary。

Go Driver 事务选项可以配置 read concern 和 write concern，但不要为了“看起来性能好”随便从 secondary 做强一致业务读。

## 5. Go 中设置事务选项

```go
txnOpts := options.Transaction().
    SetReadConcern(readconcern.Snapshot()).
    SetWriteConcern(writeconcern.Majority())

_, err := session.WithTransaction(ctx, func(sc mongo.SessionContext) (any, error) {
    return nil, nil
}, txnOpts)
```

## 6. 业务选择建议

### 普通内容浏览

可以接受轻微延迟，未来可考虑 secondary 读。

### 订单支付

写入应考虑 majority。

支付后立即读取应从 primary。

### 后台报表

可以接受延迟，可能使用 secondary 或离线统计。

### 钱包余额

强一致要求高，谨慎使用弱一致读。

## 7. 不要过早复杂化

学习阶段不要在每个操作里随便设置 read/write concern。

先掌握：

- 默认行为。
- 重要写入为什么要 majority。
- secondary 读可能读旧数据。
- 事务中的 snapshot 和 majority。

后面复制集阶段再系统实验。

## 8. 本节练习

判断：

1. 用户浏览文章列表，是否必须 primary 读？
2. 用户支付成功后查询订单，是否适合 secondary 读？
3. 钱包转账事务，writeConcern 应该更偏向什么？
4. 后台日报统计，是否能接受稍旧数据？
5. 为什么 secondary 读可能带来一致性问题？

## 9. 本节小结

你需要记住：

- writeConcern 控制写入确认级别。
- readConcern 控制读取一致性级别。
- readPreference 控制从哪个节点读。
- 重要写入常考虑 majority。
- secondary 读可能读到旧数据。
- 事务通常结合 snapshot 和 majority。

## 10. 三者分别回答什么问题

这三个概念经常混淆，可以这样记：

```text
writeConcern：我写入后，要等多少节点确认？
readConcern：我读取时，想读到什么一致性级别的数据？
readPreference：我这次读取，要从哪个节点读？
```

示例：

```text
订单创建：
  writeConcern = majority
  readPreference = primary

后台统计：
  writeConcern = 默认或按业务
  readPreference = secondaryPreferred

事务转账：
  readConcern = snapshot
  writeConcern = majority
```

## 11. 错误搭配示例

### 支付成功后读 secondary

```text
支付回调写入 primary 成功
用户立刻查询订单
查询走 secondary
secondary 复制延迟
用户看到未支付
```

这种体验会让用户重复支付或联系客服。支付后的关键查询应该优先读 primary。

### 关键写入用 w:1 后立刻宕机

`w:1` 只代表 primary 确认。primary 确认后如果还没复制到多数节点就故障，极端情况下可能被回滚。

对订单、钱包、支付流水这类数据，更应该考虑 majority。

### 全局 secondaryPreferred

如果客户端全局设置 secondaryPreferred，所有读取都可能读旧，包括：

- 登录态。
- 权限。
- 订单状态。
- 钱包余额。

更合理的方式是按集合或按 Repository 方法设置。

## 12. Go 中按集合配置

```go
orders := db.Collection("orders", options.Collection().
    SetReadPreference(readpref.Primary()).
    SetReadConcern(readconcern.Majority()).
    SetWriteConcern(writeconcern.Majority()))
```

报表集合：

```go
events := db.Collection("events", options.Collection().
    SetReadPreference(readpref.SecondaryPreferred()))
```

事务中：

```go
txnOpts := options.Transaction().
    SetReadConcern(readconcern.Snapshot()).
    SetWriteConcern(writeconcern.Majority())
```

这些配置要和业务风险匹配，而不是为了统一而统一。

## 13. 业务选择表

| 业务 | writeConcern | readConcern | readPreference | 原因 |
| --- | --- | --- | --- | --- |
| 钱包转账 | majority | snapshot/majority | primary | 金额强一致 |
| 订单创建 | majority | majority | primary | 核心业务 |
| 支付回调 | majority | majority | primary | 防重复和状态可靠 |
| 浏览量 | w:1/异步 | local | primary/secondary | 可最终一致 |
| 后台报表 | 视写入 | local/majority | secondaryPreferred | 可接受延迟 |

这张表不是死规则，真实项目要结合 RPO、延迟、成本和用户体验决策。

## 14. 和事务的关系

事务里常用：

```text
readConcern: snapshot
writeConcern: majority
readPreference: primary
```

原因：

- snapshot 保证事务内一致视图。
- majority 提高提交可靠性。
- 事务写入需要 primary。

不要在事务里做 secondary 读来追求性能。事务的目标是强一致，不是读扩展。

## 15. 设计练习

为下面接口填写参数：

```text
POST /wallet/transfer
POST /orders
GET /orders/{id} after payment
GET /articles/{id}
POST /articles/{id}/view
GET /admin/daily-report
```

格式：

```text
接口：
writeConcern：
readConcern：
readPreference：
是否允许读旧：
是否需要幂等：
理由：
```

## 16. 官方文档延伸阅读

- [Read Concern](https://www.mongodb.com/docs/manual/reference/read-concern/)
- [Write Concern](https://www.mongodb.com/docs/manual/reference/write-concern/)
- [Read Preference](https://www.mongodb.com/docs/manual/core/read-preference/)
