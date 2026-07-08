# 01 Hook、Scope 与 Context

本节目标：围绕「hook scope context」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

高级特性不是为了炫技，而是为了减少重复、统一规则和增强可维护性。

## 1. Hook 是什么

Hook 是模型生命周期钩子。例如创建前、创建后、更新前、删除前。

```go
func (u *User) BeforeCreate(tx *gorm.DB) error {
    if u.Username == "" {
        return fmt.Errorf("username required")
    }
    return nil
}
```

常见 Hook：

- `BeforeCreate`
- `AfterCreate`
- `BeforeUpdate`
- `AfterUpdate`
- `BeforeDelete`
- `AfterDelete`
- `AfterFind`

## 2. Hook 适合做什么

适合：

- 自动生成 UUID。
- 创建前填充默认业务字段。
- 统一校验模型内部不变量。
- 写入审计字段。

不适合：

- 复杂业务流程。
- 调用外部服务。
- 发送短信邮件。
- 做很重的查询。

复杂业务应该放在 Service 层。

## 3. Scope 是什么

Scope 是可复用查询条件。

分页 Scope：

```go
func Paginate(page, pageSize int) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if page < 1 {
            page = 1
        }
        if pageSize < 1 {
            pageSize = 10
        }
        if pageSize > 100 {
            pageSize = 100
        }

        offset := (page - 1) * pageSize
        return db.Offset(offset).Limit(pageSize)
    }
}
```

使用：

```go
db.Scopes(Paginate(page, pageSize)).
    Where("status = ?", "published").
    Find(&articles)
```

## 4. 业务过滤 Scope

只查已发布文章：

```go
func Published(db *gorm.DB) *gorm.DB {
    return db.Where("status = ?", "published")
}
```

使用：

```go
db.Scopes(Published, Paginate(page, pageSize)).Find(&articles)
```

## 5. Context

GORM 支持传入 `context.Context`：

```go
err := db.WithContext(ctx).
    Where("id = ?", id).
    First(&user).Error
```

在 Web 项目中，应该从请求中传递 context：

```go
repo.FindByID(c.Request.Context(), id)
```

Context 可以用于：

- 请求取消。
- 超时控制。
- 链路追踪。
- 日志字段传递。

## 6. 超时控制

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

err := db.WithContext(ctx).Find(&articles).Error
```

当查询超过超时时间，context 会取消。

## 本节练习

- [ ] 写一个 `BeforeCreate` 自动生成 UUID。
- [ ] 写一个分页 Scope。
- [ ] 写一个已发布文章 Scope。
- [ ] Repository 方法全部接收 `context.Context`。
- [ ] 给一个慢查询加 2 秒超时。
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

这一节不要只停留在“看懂概念”。建议你按下面的方式把「hook scope context」练成可以在项目中使用的能力。

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
