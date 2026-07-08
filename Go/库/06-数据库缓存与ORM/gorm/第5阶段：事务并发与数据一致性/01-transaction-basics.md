# 01 事务基础与 GORM 写法

本节目标：围绕「transaction basics」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

事务用于保证一组数据库操作要么全部成功，要么全部失败。只要业务里出现“多个写操作必须保持一致”，你就应该想到事务。

## 1. 什么时候需要事务

典型场景：

- 下单：创建订单、创建明细、扣库存。
- 支付：创建支付记录、修改订单状态。
- 转账：扣减 A 账户、增加 B 账户。
- 编辑文章：更新文章、替换标签。
- 删除用户：删除用户、清理关联数据。

如果这些步骤中某一步失败，前面的步骤也应该回滚。

## 2. GORM 闭包事务

推荐优先使用闭包事务：

```go
err := db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&order).Error; err != nil {
        return err
    }

    if err := tx.Create(&orderItems).Error; err != nil {
        return err
    }

    return nil
})
```

规则：

- 返回 `nil`：提交事务。
- 返回非 nil 错误：回滚事务。
- 事务内部必须使用 `tx`，不要继续使用外面的 `db`。

## 3. 手动事务

```go
tx := db.Begin()
if tx.Error != nil {
    return tx.Error
}

if err := tx.Create(&order).Error; err != nil {
    tx.Rollback()
    return err
}

if err := tx.Create(&orderItems).Error; err != nil {
    tx.Rollback()
    return err
}

if err := tx.Commit().Error; err != nil {
    return err
}
```

手动事务更容易漏掉回滚，所以普通业务里更推荐闭包事务。

## 4. 事务里的错误处理

不要吞掉错误：

```go
err := db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&order).Error; err != nil {
        return err
    }

    return nil
})
```

错误示例：

```go
db.Transaction(func(tx *gorm.DB) error {
    tx.Create(&order)
    return nil
})
```

这样即使创建失败，也可能提交事务。

## 5. 嵌套事务和保存点

GORM 支持嵌套事务和保存点，但初学阶段先掌握普通事务。

保存点大致用于：

```go
tx.SavePoint("sp1")
tx.RollbackTo("sp1")
```

适合复杂业务中局部回滚。一般 CRUD 项目很少用。

## 6. 默认事务

GORM 默认会在写操作时使用事务来保证数据一致性。这会带来一定性能开销。

某些高性能批量写入场景，可以考虑关闭默认事务，但前提是你清楚业务风险：

```go
db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
    SkipDefaultTransaction: true,
})
```

学习阶段不建议过早关闭。

## 本节练习

- [ ] 使用闭包事务创建一篇文章并关联标签。
- [ ] 故意让标签关联失败，确认文章是否回滚。
- [ ] 用手动事务重写同样逻辑。
- [ ] 在事务内部故意使用外部 `db`，观察为什么这是危险写法。

## 小结

事务代码记住三句话：

```text
多步写入要用事务
事务内部只用 tx
错误必须 return 出去
```
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
