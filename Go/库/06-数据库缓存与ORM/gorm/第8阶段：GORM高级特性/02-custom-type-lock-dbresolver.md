# 02 自定义类型、锁与多数据库

本节目标：围绕「custom type lock dbresolver」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

复杂项目里，你会遇到 JSON 字段、锁、多数据库连接、读写分离等需求。本节先建立基本认识。

## 1. JSON 自定义字段

例如文章扩展信息：

```go
type ArticleMeta struct {
    CoverURL string   `json:"cover_url"`
    Keywords []string `json:"keywords"`
}
```

可以使用 GORM Serializer：

```go
type Article struct {
    ID   uint
    Meta ArticleMeta `gorm:"serializer:json"`
}
```

这样 `Meta` 会以 JSON 形式存入数据库字段。

## 2. 悲观锁

```go
err := tx.Clauses(clause.Locking{Strength: "UPDATE"}).
    First(&product, productID).Error
```

适合在事务中锁住关键行，避免其他事务同时修改。

注意：

- 必须在事务中使用。
- 锁持有时间要短。
- 要避免死锁。

## 3. 乐观锁

增加 `Version` 字段：

```go
type Product struct {
    ID      uint
    Stock   int
    Version int
}
```

更新时检查版本：

```go
result := db.Model(&Product{}).
    Where("id = ? AND version = ?", product.ID, product.Version).
    Updates(map[string]any{
        "stock":   newStock,
        "version": gorm.Expr("version + 1"),
    })
```

`RowsAffected == 0` 表示冲突。

## 4. 多数据库连接

有些项目会区分：

- 业务库
- 日志库
- 报表库

最简单方式是创建多个 `*gorm.DB`：

```go
bizDB, err := database.New(bizDSN)
logDB, err := database.New(logDSN)
```

然后分别传入不同 Repository。

## 5. 读写分离

GORM 提供 Database Resolver 插件支持读写分离、多数据源等能力。

基本思路：

- 写操作走主库。
- 读操作走从库。
- 事务通常走主库。

读写分离不是初学阶段刚需，但你要知道它解决的是数据库扩展能力问题。

## 6. 分库分表

分库分表是更复杂的架构问题，不是简单加一个 GORM 配置就结束。

你需要考虑：

- 分片键怎么选。
- 跨分片查询怎么办。
- 全局唯一 ID 怎么生成。
- 事务边界怎么处理。
- 运维和迁移成本。

初中级后端先把单库模型、索引、事务学扎实。

## 本节练习

- [ ] 给文章增加 JSON 扩展字段。
- [ ] 写一个悲观锁读取商品的示例。
- [ ] 写一个乐观锁更新库存的示例。
- [ ] 创建两个数据库连接，分别代表业务库和日志库。
- [ ] 阅读 GORM Database Resolver 的官方说明。
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

这一节不要只停留在“看懂概念”。建议你按下面的方式把「custom type lock dbresolver」练成可以在项目中使用的能力。

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
