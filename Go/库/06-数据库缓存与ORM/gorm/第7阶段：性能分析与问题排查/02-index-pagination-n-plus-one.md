# 02 索引、分页与 N+1 优化

本节目标：围绕「index pagination n plus one」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

大部分 GORM 性能问题来自三类：索引不合理、分页方式粗糙、关联查询写出了 N+1。

## 1. 常用索引设计

博客系统常见索引：

```go
type Article struct {
    ID         uint
    UserID     uint      `gorm:"index"`
    CategoryID uint      `gorm:"index:idx_category_status_created"`
    Status     string    `gorm:"index:idx_category_status_created"`
    CreatedAt  time.Time `gorm:"index:idx_category_status_created"`
}
```

这个联合索引适合：

```sql
WHERE category_id = ? AND status = ?
ORDER BY created_at DESC
```

## 2. 不要乱加索引

索引会提升查询，但也会增加写入成本和存储成本。

适合加索引：

- 高频过滤字段。
- 高频排序字段。
- 外键字段。
- 唯一约束字段。

不适合乱加索引：

- 很少查询的字段。
- 区分度很低的字段单独建索引。
- 大文本字段。

## 3. 普通分页问题

```go
db.Order("created_at DESC").
    Limit(20).
    Offset(100000).
    Find(&articles)
```

大 offset 会变慢，因为数据库要跳过很多行。

## 4. 游标分页

使用上一页最后一条记录作为游标：

```go
query := db.Where("status = ?", "published")

if lastID > 0 {
    query = query.Where("id < ?", lastID)
}

err := query.Order("id DESC").
    Limit(pageSize).
    Find(&articles).Error
```

适合信息流、时间线、文章列表等“下一页”场景。

缺点：不适合直接跳到第 100 页。

## 5. 优化列表字段

文章列表不要返回正文：

```go
db.Model(&Article{}).
    Select("id", "title", "summary", "user_id", "category_id", "created_at").
    Where("status = ?", "published").
    Order("created_at DESC").
    Limit(pageSize).
    Find(&articles)
```

## 6. N+1 优化

错误写法：

```go
for _, article := range articles {
    db.First(&article.User, article.UserID)
}
```

正确写法：

```go
db.Preload("User").
    Preload("Category").
    Where("status = ?", "published").
    Limit(pageSize).
    Find(&articles)
```

## 7. 预加载也要克制

列表页通常只需要：

- 作者
- 分类

不一定需要：

- 正文
- 所有评论
- 评论用户
- 所有标签详情

详情页再加载更多关联。

## 本节练习

- [ ] 为文章列表查询设计联合索引。
- [ ] 把文章列表改成只查必要字段。
- [ ] 实现 offset 分页。
- [ ] 实现游标分页。
- [ ] 找出文章详情里的 N+1 查询。
- [ ] 用 `Preload` 优化 N+1。
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

这一节不要只停留在“看懂概念”。建议你按下面的方式把「index pagination n plus one」练成可以在项目中使用的能力。

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
