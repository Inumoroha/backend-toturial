# 01 SQL 日志与慢查询排查

本节目标：围绕「sql log slow query」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

性能优化的第一步不是猜，而是看 SQL。GORM 链式 API 写得再漂亮，如果生成的 SQL 很差，接口照样会慢。

## 1. 全局开启 SQL 日志

```go
db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
    Logger: logger.Default.LogMode(logger.Info),
})
```

学习阶段建议开 `Info`，生产环境要根据日志量谨慎配置。

## 2. 单次查询 Debug

```go
db.Debug().
    Where("status = ?", "published").
    Order("created_at DESC").
    Limit(10).
    Find(&articles)
```

适合临时排查某个查询。

## 3. 慢查询日志

自定义 logger 可以设置慢查询阈值：

```go
newLogger := logger.New(
    log.New(os.Stdout, "\r\n", log.LstdFlags),
    logger.Config{
        SlowThreshold: time.Second,
        LogLevel:      logger.Warn,
        Colorful:      true,
    },
)

db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
    Logger: newLogger,
})
```

超过阈值的 SQL 会被标记出来。

## 4. 观察 SQL 的重点

看到 SQL 后，先看这些点：

- 是否查询了不需要的字段。
- 是否没有 `WHERE`。
- 是否没有分页。
- 是否出现大量重复查询。
- 是否条件字段没有索引。
- 是否 `LIKE '%keyword%'` 导致索引难以使用。

## 5. EXPLAIN

MySQL 中可以用：

```sql
EXPLAIN SELECT * FROM articles WHERE status = 'published' ORDER BY created_at DESC LIMIT 10;
```

重点关注：

- 是否使用索引。
- 扫描行数是否过大。
- 是否出现 filesort。
- 是否出现临时表。

初学阶段不需要精通执行计划，但要养成慢 SQL 用 `EXPLAIN` 看一眼的习惯。

## 6. 常见慢查询来源

- 文章列表没有分页。
- 文章详情循环查评论用户，造成 N+1。
- 模糊搜索 `%keyword%` 数据量大后变慢。
- 排序字段没有索引。
- 统计接口每次都全表聚合。
- 返回了大字段，例如列表页返回正文。

## 本节练习

- [ ] 给博客 API 开启 SQL 日志。
- [ ] 访问文章列表，复制生成的 SQL。
- [ ] 对文章列表 SQL 执行 `EXPLAIN`。
- [ ] 找出是否使用索引。
- [ ] 故意写一个无分页查询，观察 SQL。
- [ ] 故意写一个 N+1 查询，观察重复 SQL。
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

这一节不要只停留在“看懂概念”。建议你按下面的方式把「sql log slow query」练成可以在项目中使用的能力。

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
