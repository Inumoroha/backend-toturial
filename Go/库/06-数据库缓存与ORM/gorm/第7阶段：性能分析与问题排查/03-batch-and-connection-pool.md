# 03 批量操作与连接池

本节目标：围绕「batch and connection pool」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

当数据量变大后，一条一条写入会很慢。连接池配置不合理，也会让服务在高并发时不稳定。

## 1. 批量插入

```go
users := []User{
    {Username: "u1", Email: "u1@example.com"},
    {Username: "u2", Email: "u2@example.com"},
    {Username: "u3", Email: "u3@example.com"},
}

err := db.Create(&users).Error
```

GORM 会尽量使用批量插入。

## 2. 指定批大小

```go
err := db.CreateInBatches(users, 100).Error
```

适合大量数据导入。

批大小不是越大越好，要结合数据库参数、字段数量、网络和事务大小测试。

## 3. 批量更新

把某个分类下草稿文章全部发布：

```go
err := db.Model(&Article{}).
    Where("category_id = ? AND status = ?", categoryID, "draft").
    Updates(map[string]any{
        "status": "published",
    }).Error
```

注意：批量更新必须带条件。GORM 默认会阻止没有条件的全表更新。

## 4. 使用表达式更新

阅读量加 1：

```go
err := db.Model(&Article{}).
    Where("id = ?", articleID).
    Update("view_count", gorm.Expr("view_count + ?", 1)).Error
```

库存扣减：

```go
err := db.Model(&Product{}).
    Where("id = ? AND stock >= ?", productID, quantity).
    Update("stock", gorm.Expr("stock - ?", quantity)).Error
```

## 5. 连接池配置

```go
sqlDB, err := db.DB()
if err != nil {
    return err
}

sqlDB.SetMaxOpenConns(100)
sqlDB.SetMaxIdleConns(10)
sqlDB.SetConnMaxLifetime(time.Hour)
sqlDB.SetConnMaxIdleTime(10 * time.Minute)
```

含义：

- `MaxOpenConns`：最大打开连接数。
- `MaxIdleConns`：最大空闲连接数。
- `ConnMaxLifetime`：连接最大存活时间。
- `ConnMaxIdleTime`：连接最大空闲时间。

## 6. 连接池配置建议

学习阶段：

```text
MaxOpenConns: 20
MaxIdleConns: 5
```

小型项目：

```text
MaxOpenConns: 50 - 100
MaxIdleConns: 10 - 20
```

真实生产环境要根据：

- 数据库最大连接数
- 服务实例数量
- 接口并发量
- SQL 平均耗时

共同决定。

## 本节练习

- [ ] 批量插入 100 个标签。
- [ ] 使用 `CreateInBatches` 批量插入用户。
- [ ] 批量发布某个分类下的文章。
- [ ] 使用表达式给文章阅读量加 1。
- [ ] 配置连接池参数。
- [ ] 压测前后观察数据库连接数。
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。

---

## 颗粒度补强：本节应该如何真正练会

这一节不要只停留在“看懂概念”。建议你按下面的方式把「batch and connection pool」练成可以在项目中使用的能力。

### 1. 先写最小可运行示例

在 `code/gorm-study` 中准备一个最小 Go 程序，只保留和本节主题直接相关的模型、数据库连接和 GORM 调用。不要一开始就放进完整项目结构，否则你很难判断问题来自 GORM、分层、路由还是参数绑定。

建议流程：

```text
1. 定义最小模型
2. AutoMigrate 建表
3. 插入 2 到 3 条测试数据
4. 执行本节核心 GORM API
5. 打印 SQL 日志
6. 到数据库客户端里验证结果
```

### 2. 必须观察 SQL

学习 GORM 的关键不是背 API，而是看它生成的 SQL。执行本节代码时，建议临时开启：

```go
db = db.Debug()
```

或者在初始化时使用：

```go
Logger: logger.Default.LogMode(logger.Info)
```

你至少要回答：

```text
这段 GORM 代码生成了几条 SQL？
SQL 中有没有 WHERE 条件？
有没有 LIMIT 或 ORDER BY？
有没有使用软删除条件 deleted_at IS NULL？
更新或删除时 RowsAffected 是多少？
```

### 3. 用数据库客户端验证

不要只相信 Go 程序输出。每次运行后，都在 MySQL 客户端中执行对应查询，例如：

```sql
SHOW TABLES;
SHOW CREATE TABLE users;
SHOW INDEX FROM users;
SELECT * FROM users ORDER BY id DESC LIMIT 10;
```

如果本节涉及文章、评论、订单或标签，就去查对应表：

```sql
SELECT * FROM articles ORDER BY id DESC LIMIT 10;
SELECT * FROM comments ORDER BY id DESC LIMIT 10;
SELECT * FROM orders ORDER BY id DESC LIMIT 10;
SELECT * FROM article_tags ORDER BY article_id DESC LIMIT 10;
```

### 4. 主动制造一个错误

每节都建议故意制造一个小错误，然后观察报错和数据变化。比如：

```text
把数据库密码写错，观察连接错误。
把唯一字段重复插入，观察唯一索引错误。
把查询 ID 改成不存在的值，观察 ErrRecordNotFound。
把更新条件去掉，观察 GORM 是否阻止全表更新。
把事务里的 return err 去掉，观察是否错误提交。
```

你真正排查过错误，才算把这个知识点学稳。

### 5. 放回真实项目场景

最后把本节知识放回博客系统或订单系统中思考：

```text
博客系统里哪个接口会用到这个知识点？
订单系统里哪个流程会因为这个知识点写错而出问题？
这个知识点应该放在 Model、Repository、Service 还是 Handler？
它需要测试吗？需要看 SQL 日志吗？需要事务吗？
```

如果你能回答这些问题，说明你已经从“知道 API”进入到“能做后端项目”的层次。

### 6. 本节复盘问题

学完后请写下这几个答案：

```text
1. 本节最核心的 GORM API 是什么？
2. 它生成的 SQL 大致是什么？
3. 它最容易踩的坑是什么？
4. 如果不用 GORM，原生 SQL 会怎么写？
5. 在博客系统或订单系统中，它会出现在哪个模块？
```
