# 06. readConcern 与一致性

readConcern 控制读取数据的一致性级别。它和 writeConcern、readPreference 一起决定“读到什么数据、从哪里读、写入确认到哪里”。

## 1. readConcern 解决什么问题

readConcern 关注：

```text
读操作能看到什么级别的数据？
```

例如：

- 是否能读到尚未多数确认的数据？
- 事务中是否基于一致性快照读取？
- 读到的数据是否足够持久？

## 2. 常见 readConcern

常见级别包括：

```text
local
majority
snapshot
```

学习阶段先掌握这三个。

## 3. local

`local` 表示读取当前节点可用的数据。

优点：

- 延迟低。

风险：

- 在故障切换场景中，可能读到后续会回滚的数据。

## 4. majority

`majority` 表示读取已经被多数节点确认的数据。

适合：

- 需要更强一致性的读。
- 和 majority 写入配合。

代价：

- 延迟和资源成本可能更高。

## 5. snapshot

`snapshot` 常用于事务。

它让事务在一致性快照上读取数据。

在 Go 事务选项中：

```go
txnOpts := options.Transaction().
    SetReadConcern(readconcern.Snapshot())
```

## 6. readConcern 和 readPreference 的区别

readConcern：

```text
读什么一致性级别的数据？
```

readPreference：

```text
从哪个节点读？
```

例子：

```text
从 secondary 读 majority 数据
```

这同时涉及 readPreference 和 readConcern。

## 7. 业务选择建议

### 钱包余额

建议强一致路径，谨慎使用弱一致读。

### 订单支付后状态

建议 primary + 合理 readConcern。

### 内容浏览

可以接受较弱一致性。

### 报表

可以根据实时性要求选择。

## 8. 不要过早微调

大多数学习和普通业务先使用默认设置。

重要的是你知道：

- 什么时候默认不够。
- 为什么 majority 更强。
- 为什么 secondary 读可能旧。
- 为什么事务用 snapshot。

## 9. 本节练习

回答：

1. readConcern 和 writeConcern 分别控制什么？
2. readConcern 和 readPreference 有什么区别？
3. `snapshot` 常用于什么场景？
4. 为什么强一致业务不应随便读旧数据？
5. 哪些场景可以接受 eventual consistency？

## 10. 本节小结

你需要记住：

- readConcern 控制读取一致性级别。
- writeConcern 控制写入确认级别。
- readPreference 控制从哪个节点读。
- 事务中常见 snapshot。
- 一致性选择是业务决策，不只是数据库参数。

## 11. 三个参数不要混淆

一致性相关参数经常被混在一起：

```text
readConcern：读到什么一致性级别的数据
writeConcern：写入要被多少节点确认
readPreference：从哪个节点读
```

它们分别回答不同问题。

示例：

```text
writeConcern majority
  -> 写入被多数节点确认

readPreference secondary
  -> 读请求可以去 secondary

readConcern majority
  -> 读取多数派确认过的数据
```

不能因为设置了 `readPreference=secondary`，就认为读到的一定是最新数据。secondary 可能存在复制延迟。

## 12. 典型业务选择

### 强一致业务

例如：

- 钱包余额。
- 订单支付状态。
- 库存扣减结果。
- 权益发放。

建议：

```text
写：majority
读：primary
必要时使用事务
```

这样牺牲一些延迟，换取更清晰的一致性。

### 可接受短暂延迟的业务

例如：

- 文章浏览量。
- 用户动态展示。
- 后台报表。
- 内容推荐特征。

可以考虑：

```text
读：secondaryPreferred
统计：异步或离线
```

但必须在产品上接受“刚写完不一定马上读到”的体验。

## 13. Go Driver 中设置 readConcern

集合级别设置：

```go
coll := db.Collection("orders", options.Collection().
    SetReadConcern(readconcern.Majority()).
    SetWriteConcern(writeconcern.Majority()))
```

客户端级别设置：

```go
client, err := mongo.Connect(options.Client().
    ApplyURI(uri).
    SetReadConcern(readconcern.Majority()).
    SetWriteConcern(writeconcern.Majority()))
```

读偏好示例：

```go
analytics := db.Collection("events", options.Collection().
    SetReadPreference(readpref.SecondaryPreferred()))
```

不要为了“提升性能”全局设置 secondary 读。不同集合、不同接口的一致性要求不一样。

## 14. 一致性 Bug 示例

用户支付成功后，服务写入：

```text
orders.status = paid
payments.status = success
```

如果订单详情页立刻从 secondary 读，而 secondary 复制延迟 2 秒，用户可能看到：

```text
支付成功，但订单仍显示待支付
```

这不是页面 Bug，而是一致性选择导致的现象。

解决思路：

- 支付后关键读走 primary。
- 订单状态更新使用 majority。
- 支付状态和订单状态设计成可幂等修正。
- 前端可短暂展示“处理中”，但后端状态必须可靠。

## 15. 事务中的 snapshot

事务中常见 `snapshot` 读关注，它用于让事务内部读取一个一致视图。

例如转账：

```text
读取 A 余额
读取 B 余额
扣减 A
增加 B
写流水
```

事务内部不希望读到一半时外部写入改变了视图。`snapshot` 能帮助事务在一致视图上执行。

但事务不是免费的：

- 时间不能太长。
- 不要做网络调用。
- 不要在事务中做大范围扫描。
- 回调需要能安全重试。

## 16. 一致性决策模板

```md
# 一致性设计

## 业务接口

## 数据风险

- 金额：
- 库存：
- 权益：
- 展示：

## 写入关注

## 读取来源

## 是否允许读旧

## 最大可接受延迟

## 是否需要事务

## 失败重试策略
```

## 17. 官方文档延伸阅读

- [Read Concern](https://www.mongodb.com/docs/manual/reference/read-concern/)
- [Write Concern](https://www.mongodb.com/docs/manual/reference/write-concern/)
- [Read Preference](https://www.mongodb.com/docs/manual/core/read-preference/)
