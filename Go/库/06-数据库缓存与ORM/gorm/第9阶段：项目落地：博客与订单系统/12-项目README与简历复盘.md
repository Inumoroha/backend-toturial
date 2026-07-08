# 12 项目 README 与简历复盘

本节目标：把项目整理成可以展示、可以复盘、可以写进简历的状态。学习项目不是写完代码就结束，还要能讲清楚。

---

## 一、README 应包含

```text
项目介绍
技术栈
功能列表
目录结构
启动方式
环境变量
数据库迁移
接口说明
测试方式
```

---

## 二、启动说明示例

```powershell
docker compose up -d
go run ./cmd/server
```

环境变量：

```text
DB_DSN=...
JWT_SECRET=...
APP_PORT=8080
```

---

## 三、项目亮点

博客系统：

- GORM 关联关系。
- 多对多标签。
- Preload 优化详情。
- 事务保证文章和标签一致。
- 分页和索引优化。

订单系统：

- 事务下单。
- 条件更新扣库存。
- RowsAffected 防并发冲突。
- 状态机防重复支付。
- 回滚测试。

---

## 四、简历描述

```text
基于 Go、Gin、GORM、MySQL 实现博客和订单系统，使用 GORM 完成模型映射、关联查询、事务控制和性能优化。博客模块支持用户认证、文章标签多对多、评论软删除和分页查询；订单模块使用事务保证订单、订单明细和库存一致性，通过条件更新和 RowsAffected 防止库存超扣。
```

---

## 五、复盘问题

- 为什么 Model 和 DTO 要分离？
- 为什么创建文章要用事务？
- 为什么扣库存要带条件？
- 如何避免 N+1？
- 如何设计文章列表索引？
- 如何处理重复支付？

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 写出完整项目 README。
- 总结项目技术亮点。
- 把 GORM 能力讲成项目经验。
- 准备面试复盘材料。

---

## 颗粒度补强：本节应该如何真正练会

这一节不要只停留在“看懂概念”。建议你按下面的方式把「项目README与简历复盘」练成可以在项目中使用的能力。

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
