# 06. Go Driver v2 事务写法

本节学习 Go Driver v2 中的事务写法。核心流程是：创建 session，调用 `WithTransaction`，在回调中使用 `mongo.SessionContext` 执行所有事务内操作。

## 1. 基本结构

```go
session, err := client.StartSession()
if err != nil {
    return err
}
defer session.EndSession(ctx)

_, err = session.WithTransaction(ctx, func(sc mongo.SessionContext) (any, error) {
    // 所有事务内操作都使用 sc
    return nil, nil
})
if err != nil {
    return err
}
```

要点：

- `StartSession()` 创建 session。
- `WithTransaction()` 开启、提交、必要时中止事务。
- 回调参数 `sc` 是事务上下文。
- 事务内数据库操作必须使用 `sc`，不要用外层 `ctx`。

## 2. 转账函数示例

```go
type TransferInput struct {
    TransactionNo string
    FromWalletID  bson.ObjectID
    ToWalletID    bson.ObjectID
    Amount        int64
}

func Transfer(ctx context.Context, client *mongo.Client, db *mongo.Database, input TransferInput) error {
    if input.Amount <= 0 {
        return ErrInvalidAmount
    }

    session, err := client.StartSession()
    if err != nil {
        return err
    }
    defer session.EndSession(ctx)

    wallets := db.Collection("wallets")
    txs := db.Collection("wallet_transactions")

    _, err = session.WithTransaction(ctx, func(sc mongo.SessionContext) (any, error) {
        now := time.Now().UTC()

        deductResult, err := wallets.UpdateOne(sc,
            bson.D{
                {"_id", input.FromWalletID},
                {"balance", bson.D{{"$gte", input.Amount}}},
            },
            bson.D{
                {"$inc", bson.D{{"balance", -input.Amount}}},
                {"$set", bson.D{{"updated_at", now}}},
            },
        )
        if err != nil {
            return nil, err
        }
        if deductResult.MatchedCount == 0 {
            return nil, ErrInsufficientBalance
        }

        addResult, err := wallets.UpdateOne(sc,
            bson.D{{"_id", input.ToWalletID}},
            bson.D{
                {"$inc", bson.D{{"balance", input.Amount}}},
                {"$set", bson.D{{"updated_at", now}}},
            },
        )
        if err != nil {
            return nil, err
        }
        if addResult.MatchedCount == 0 {
            return nil, ErrWalletNotFound
        }

        _, err = txs.InsertOne(sc, bson.D{
            {"transaction_no", input.TransactionNo},
            {"from_wallet_id", input.FromWalletID},
            {"to_wallet_id", input.ToWalletID},
            {"amount", input.Amount},
            {"type", "transfer"},
            {"status", "success"},
            {"created_at", now},
        })
        if err != nil {
            if mongo.IsDuplicateKeyError(err) {
                return nil, ErrDuplicateTransaction
            }
            return nil, err
        }

        return nil, nil
    })

    return err
}
```

需要导入：

```go
import (
    "context"
    "errors"
    "time"

    "go.mongodb.org/mongo-driver/v2/bson"
    "go.mongodb.org/mongo-driver/v2/mongo"
)
```

## 3. 事务错误定义

```go
var (
    ErrInvalidAmount        = errors.New("invalid amount")
    ErrInsufficientBalance  = errors.New("insufficient balance")
    ErrWalletNotFound       = errors.New("wallet not found")
    ErrDuplicateTransaction = errors.New("duplicate transaction")
)
```

## 4. 事务选项

可以设置事务选项：

```go
txnOpts := options.Transaction().
    SetReadConcern(readconcern.Snapshot()).
    SetWriteConcern(writeconcern.Majority())

_, err = session.WithTransaction(ctx, callback, txnOpts)
```

需要导入：

```go
"go.mongodb.org/mongo-driver/v2/mongo/options"
"go.mongodb.org/mongo-driver/v2/mongo/readconcern"
"go.mongodb.org/mongo-driver/v2/mongo/writeconcern"
```

学习阶段先理解即可，后面第 7 阶段复制集会更深入。

## 5. 事务内不要并行操作

不要这样：

```go
_, err = session.WithTransaction(ctx, func(sc mongo.SessionContext) (any, error) {
    go wallets.UpdateOne(sc, ...)
    go txs.InsertOne(sc, ...)
    return nil, nil
})
```

官方 Go Driver 文档明确提醒：单个事务内不支持并行操作。

事务内按顺序执行。

## 6. 回调要能安全重试

`WithTransaction` 可能在特定错误下重试提交或事务。

因此回调中不要做不可回滚副作用：

- 发送短信。
- 调用支付扣款。
- 发送消息给外部系统。
- 写本地文件。

如果必须做外部副作用，通常先在事务内写 outbox/event 表，事务提交后再异步发送。

## 7. 本节练习

完成：

1. 定义 `TransferInput`。
2. 定义转账错误。
3. 实现 `Transfer` 函数。
4. 在事务中扣款、加款、写流水。
5. 所有事务内操作使用 `mongo.SessionContext`。
6. 处理余额不足、钱包不存在、重复交易号。
7. 写出为什么事务回调里不能发短信。

## 8. 本节小结

你需要记住：

- Go Driver v2 用 `StartSession` 和 `WithTransaction`。
- 事务内操作使用 `mongo.SessionContext`。
- 事务内不要并行执行操作。
- 回调要能安全重试。
- 事务选项可设置 readConcern 和 writeConcern。

