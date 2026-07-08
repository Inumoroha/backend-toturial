# 08 Preload 高级用法

本节目标：掌握条件预加载、嵌套预加载、选择字段预加载和预加载的边界。`Preload` 是 GORM 关联查询的核心能力。

---

## 一、基础预加载

```go
db.Preload("User").First(&article, id)
```

加载文章作者。

---

## 二、多个关联

```go
db.Preload("User").
    Preload("Category").
    Preload("Tags").
    First(&article, id)
```

适合文章详情页。

---

## 三、嵌套预加载

评论需要评论人：

```go
db.Preload("Comments.User").
    First(&article, id)
```

含义：

```text
加载 Comments
再加载每条 Comment 的 User
```

---

## 四、条件预加载

只加载审核通过评论：

```go
db.Preload("Comments", "status = ?", "approved").
    First(&article, id)
```

---

## 五、带排序的预加载

```go
db.Preload("Comments", func(db *gorm.DB) *gorm.DB {
    return db.Order("created_at ASC")
}).First(&article, id)
```

评论一般按时间正序或倒序，要明确。

---

## 六、选择字段

```go
db.Preload("User", func(db *gorm.DB) *gorm.DB {
    return db.Select("id", "username")
}).First(&article, id)
```

不要把用户密码哈希加载出来。

---

## 七、预加载边界

不要无限嵌套：

```go
Preload("Comments.User.Articles.Comments")
```

这种查询会变得很重，也难以维护。

接口要根据页面需要设计，不要一次返回整个世界。

---

## 八、常见错误

### 1. Preload 字段名写错

必须是结构体字段名。

### 2. 忘记限制子数据

文章评论很多时，详情页一次加载全部可能很慢。

### 3. 加载敏感字段

预加载用户时注意不要暴露 `PasswordHash`，响应层也要用 DTO。

---

## 九、本节达标标准

学完本节后，你应该能够做到：

- 使用多个 `Preload`。
- 使用嵌套预加载。
- 使用条件预加载。
- 在预加载中排序。
- 理解预加载不应该无限扩展。

---

## 颗粒度补强：本节应该如何真正练会

这一节不要只停留在“看懂概念”。建议你按下面的方式把「Preload高级用法」练成可以在项目中使用的能力。

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
