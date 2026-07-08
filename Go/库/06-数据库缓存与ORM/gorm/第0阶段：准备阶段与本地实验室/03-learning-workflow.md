# 03 学习方法与练习规范

本节目标：围绕「learning workflow」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

GORM 学习最容易踩的坑是：只记 API 名字，不理解 SQL 和业务场景。本节给你一套固定学习流程，后续每个阶段都按这个节奏推进。

## 1. 每个知识点的学习流程

建议按照下面 6 步学习：

```text
1. 先理解业务问题
2. 写出原生 SQL
3. 用 GORM 实现
4. 打印 GORM 生成的 SQL
5. 对比原生 SQL 和 GORM SQL
6. 写一个小练习巩固
```

例如学习分页时，不要一上来只写：

```go
db.Limit(10).Offset(20).Find(&articles)
```

你还要知道它对应：

```sql
SELECT * FROM articles LIMIT 10 OFFSET 20;
```

## 2. 推荐项目练习方式

后续教程会围绕两个核心项目：

### 博客系统

用于练习：

- 用户
- 文章
- 分类
- 标签
- 评论
- CRUD
- 关联查询
- 分页查询

### 订单系统

用于练习：

- 用户
- 商品
- 订单
- 订单明细
- 库存扣减
- 事务
- 并发控制

这两个项目覆盖大部分后端业务基础。

## 3. 代码组织建议

刚开始学习时，先不要急着分层。可以从一个 `main.go` 开始，确认 API 和 SQL 都理解以后，再移动到工程结构里。

学习初期：

```text
gorm-study
├── go.mod
└── main.go
```

工程化阶段：

```text
gorm-study
├── cmd
│   └── server
│       └── main.go
├── internal
│   ├── database
│   ├── model
│   ├── repository
│   ├── service
│   └── handler
├── go.mod
└── go.sum
```

## 4. 每日复盘模板

每天学完后，用下面模板记录：

```markdown
## 今日主题

## 我写了哪些 GORM API

## 这些 API 生成了哪些 SQL

## 我遇到的问题

## 明天要继续补什么
```

## 5. 判断是否真正掌握

一个知识点掌握了，不是指你看懂了，而是你能做到：

- 不看资料写出最小示例。
- 能解释生成的 SQL。
- 能说出适用场景。
- 能说出常见坑。
- 能把它放进一个真实业务接口。

## 本节练习

- [ ] 建立一个 `notes` 文件夹记录每日复盘。
- [ ] 为博客系统写一份表设计草稿。
- [ ] 为订单系统写一份下单流程草稿。
- [ ] 准备一个长期使用的 `gorm-study` 项目。
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

这一节不要只停留在“看懂概念”。建议你按下面的方式把「learning workflow」练成可以在项目中使用的能力。

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
