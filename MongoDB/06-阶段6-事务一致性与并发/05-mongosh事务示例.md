# 05. mongosh 事务示例

本节用 `mongosh` 理解 session、事务、提交和回滚。注意：多文档事务需要支持事务的部署环境，通常是副本集或分片集群。

如果你当前本地 MongoDB 是 standalone，事务可能无法正常运行。第 7 阶段会搭建复制集。本节先理解写法和流程。

## 1. 准备数据

```javascript
use transaction_stage6

db.wallets.drop()
db.wallet_transactions.drop()

db.wallets.insertMany([
  {
    _id: ObjectId("671000000000000000000001"),
    user_id: ObjectId("660000000000000000000001"),
    balance: 10000,
    updated_at: new Date()
  },
  {
    _id: ObjectId("671000000000000000000002"),
    user_id: ObjectId("660000000000000000000002"),
    balance: 5000,
    updated_at: new Date()
  }
])
```

单位是分。

## 2. 创建唯一索引

```javascript
db.wallet_transactions.createIndex(
  { transaction_no: 1 },
  { unique: true }
)
```

转账流水号必须唯一，用于幂等。

## 3. 启动 session

```javascript
const session = db.getMongo().startSession()
const sessionDb = session.getDatabase("transaction_stage6")
```

## 4. 执行事务

```javascript
session.startTransaction()

try {
  const fromWalletId = ObjectId("671000000000000000000001")
  const toWalletId = ObjectId("671000000000000000000002")
  const amount = 1000
  const transactionNo = "T202607050001"
  const now = new Date()

  const deductResult = sessionDb.wallets.updateOne(
    { _id: fromWalletId, balance: { $gte: amount } },
    {
      $inc: { balance: -amount },
      $set: { updated_at: now }
    }
  )

  if (deductResult.matchedCount === 0) {
    throw new Error("余额不足")
  }

  const addResult = sessionDb.wallets.updateOne(
    { _id: toWalletId },
    {
      $inc: { balance: amount },
      $set: { updated_at: now }
    }
  )

  if (addResult.matchedCount === 0) {
    throw new Error("收款钱包不存在")
  }

  sessionDb.wallet_transactions.insertOne({
    transaction_no: transactionNo,
    from_wallet_id: fromWalletId,
    to_wallet_id: toWalletId,
    amount,
    type: "transfer",
    status: "success",
    created_at: now
  })

  session.commitTransaction()
  print("事务提交成功")
} catch (e) {
  session.abortTransaction()
  print("事务已回滚: " + e.message)
} finally {
  session.endSession()
}
```

## 5. 检查结果

```javascript
db.wallets.find().pretty()
db.wallet_transactions.find().pretty()
```

如果事务提交成功：

- A 余额减少。
- B 余额增加。
- 流水写入。

如果中途抛错：

- 三个操作都不应该生效。

## 6. 模拟失败

把转账金额改成超过余额：

```javascript
const amount = 999999
```

扣款 update 的 `matchedCount` 会是 0，然后抛出“余额不足”，事务回滚。

## 7. 为什么流水也要唯一索引

如果同一笔转账请求重复执行，事务本身不会自动知道这是重复请求。

唯一索引：

```javascript
{ transaction_no: 1 }
```

可以阻止重复流水。

业务层遇到重复键时，应查询已有流水并返回已有结果。

## 8. 本节练习

完成：

1. 准备两个钱包。
2. 创建 `transaction_no` 唯一索引。
3. 写一段事务完成转账。
4. 模拟余额不足，确认回滚。
5. 模拟重复交易号，观察唯一索引错误。
6. 写出为什么事务仍然需要幂等键。

## 9. 本节小结

你需要记住：

- 事务需要 session。
- `startTransaction` 开始事务。
- `commitTransaction` 提交事务。
- `abortTransaction` 回滚事务。
- 业务失败要主动抛错并回滚。
- 事务仍然需要唯一索引和幂等键。

