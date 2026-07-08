# 03 GORM Gen 类型安全查询

本节目标：围绕「gorm gen」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

GORM Gen 是 GORM 的代码生成工具，可以生成类型安全的查询代码。它适合较大的项目，能减少字符串字段名带来的错误。

## 1. 普通 GORM 查询的问题

```go
db.Where("emial = ?", email).First(&user)
```

这里 `email` 拼成了 `emial`，编译器发现不了，只有运行时才报错。

GORM Gen 的目标是让更多查询在编译期获得类型检查。

## 2. 适合什么时候学习 Gen

建议在你掌握这些内容后再学：

- 模型定义
- CRUD
- 条件查询
- 关联关系
- Repository 分层

也就是完成本教程前 6 个阶段后，再学习 Gen 会更自然。

## 3. 基本流程

通常流程：

```text
1. 定义 GORM Model
2. 编写生成器程序
3. 执行代码生成
4. 在 Repository 中使用生成的 query 包
```

## 4. 示例查询风格

生成代码后，查询可能类似：

```go
q := query.Use(db)
u := q.User

user, err := u.WithContext(ctx).
    Where(u.Email.Eq(email)).
    First()
```

优势：

- 字段名由代码生成。
- 查询条件更类型安全。
- 重构字段时更容易发现问题。

## 5. Gen 的取舍

适合：

- 中大型项目。
- Repository 查询很多。
- 团队希望减少运行时字段名错误。
- 希望查询代码更规范。

不一定适合：

- 刚入门学习。
- 很小的 CRUD 项目。
- 项目模型频繁试错阶段。

## 6. 推荐学习顺序

```text
先用普通 GORM 写完整项目
  -> 抽出 Repository
  -> 找到重复查询和字段名字符串
  -> 引入 GORM Gen
  -> 逐步替换关键查询
```

不要一开始就上 Gen，否则你会同时面对 ORM、代码生成、工程结构三件事，学习负担会变重。

## 本节练习

- [ ] 阅读 GORM Gen 官方文档。
- [ ] 为博客系统模型生成 query 代码。
- [ ] 用 Gen 重写按邮箱查询用户。
- [ ] 用 Gen 重写文章分页查询。
- [ ] 对比普通 GORM 和 Gen 的代码可读性。
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

这一节不要只停留在“看懂概念”。建议你按下面的方式把「gorm gen」练成可以在项目中使用的能力。

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
