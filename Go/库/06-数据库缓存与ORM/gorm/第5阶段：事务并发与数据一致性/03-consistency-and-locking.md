# 03 并发一致性与锁

本节目标：围绕「consistency and locking」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

事务能保证一组 SQL 的原子性，但并发场景还需要考虑多个请求同时修改同一份数据。例如两个用户同时购买最后一件商品。

## 1. 并发扣库存的问题

假设库存是 1：

```text
请求 A 查询库存：1
请求 B 查询库存：1
请求 A 扣减库存：成功
请求 B 扣减库存：也成功
```

如果代码只是在应用层判断库存，就可能超卖。

## 2. 条件更新是基础防线

推荐写法：

```go
result := tx.Model(&Product{}).
    Where("id = ? AND stock >= ?", productID, quantity).
    Update("stock", gorm.Expr("stock - ?", quantity))
```

数据库会保证这条更新的原子性。

如果两个请求同时扣库存，只有一个能把库存从 1 扣到 0，另一个会因为 `stock >= quantity` 不满足而影响 0 行。

## 3. 悲观锁

悲观锁的思路是：我读取这行数据时先锁住，其他事务等我处理完。

GORM 写法示例：

```go
var product Product

err := tx.Clauses(clause.Locking{Strength: "UPDATE"}).
    First(&product, productID).Error
```

需要导入：

```go
import "gorm.io/gorm/clause"
```

这通常生成类似：

```sql
SELECT * FROM products WHERE id = 1 FOR UPDATE;
```

使用悲观锁时必须在事务中。

## 4. 乐观锁

乐观锁的思路是：读取时不加锁，更新时检查版本号是否变化。

模型示例：

```go
type Product struct {
    ID      uint
    Name    string
    Stock   int
    Version int
}
```

更新时：

```go
result := tx.Model(&Product{}).
    Where("id = ? AND version = ? AND stock >= ?", product.ID, product.Version, quantity).
    Updates(map[string]any{
        "stock":   gorm.Expr("stock - ?", quantity),
        "version": gorm.Expr("version + 1"),
    })
```

如果 `RowsAffected == 0`，说明版本冲突或库存不足。

## 5. 选择哪种方式

常见选择：

- 普通库存扣减：条件更新通常已经够用。
- 强一致、高竞争资源：考虑悲观锁。
- 读多写少、冲突较少：考虑乐观锁。
- 秒杀等高并发：通常还要配合 Redis、消息队列、限流等方案，不能只靠 GORM。

## 6. 事务隔离级别

数据库有不同隔离级别，例如：

- Read Committed
- Repeatable Read
- Serializable

MySQL InnoDB 默认通常是 Repeatable Read。学习初期先理解：隔离级别影响事务之间“看见彼此修改”的方式。

真正遇到并发异常时，要结合数据库类型、隔离级别、SQL 和索引一起分析。

## 本节练习

- [ ] 用条件更新实现扣库存。
- [ ] 库存为 1 时，模拟两次扣减，确认不会扣成负数。
- [ ] 用 `FOR UPDATE` 写一个悲观锁示例。
- [ ] 用 `version` 字段写一个乐观锁示例。
- [ ] 比较三种方式的代码复杂度。

## 小结

处理并发一致性时，不要只相信应用层判断。

```text
库存判断要放进 UPDATE 条件
关键写操作要检查 RowsAffected
复杂并发要理解锁和隔离级别
```
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
