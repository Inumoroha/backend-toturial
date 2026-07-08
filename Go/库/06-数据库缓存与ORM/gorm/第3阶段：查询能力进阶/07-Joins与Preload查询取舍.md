# 07 Joins 与 Preload 查询取舍

本节目标：理解 `Joins` 和 `Preload` 的区别。它们都能处理关联数据，但适合的查询形态不同。

---

## 一、一句话区别

```text
Preload：查主表，再查关联表，组装成嵌套结构。
Joins：使用 SQL JOIN，适合扁平结果和按关联字段筛选。
```

---

## 二、Preload 适合详情页

```go
var article Article
err := db.Preload("User").
    Preload("Category").
    Preload("Tags").
    First(&article, id).Error
```

返回结构：

```go
article.User.Username
article.Category.Name
article.Tags[0].Name
```

适合文章详情这种嵌套对象。

---

## 三、Joins 适合列表 DTO

```go
type ArticleListItem struct {
    ID           uint
    Title        string
    AuthorName   string
    CategoryName string
}

var items []ArticleListItem
err := db.Table("articles AS a").
    Select("a.id, a.title, u.username AS author_name, c.name AS category_name").
    Joins("JOIN users AS u ON u.id = a.user_id").
    Joins("JOIN categories AS c ON c.id = a.category_id").
    Scan(&items).Error
```

适合列表页，因为前端只需要扁平展示。

---

## 四、按关联字段过滤

查询作者用户名为 tom 的文章：

```go
db.Table("articles AS a").
    Joins("JOIN users AS u ON u.id = a.user_id").
    Where("u.username = ?", "tom").
    Scan(&items)
```

这种场景用 `Joins` 更直接。

---

## 五、Preload 不是 JOIN

很多初学者以为 `Preload("User")` 会生成 JOIN。

实际上 GORM 通常会执行类似：

```sql
SELECT * FROM articles WHERE id = 1;
SELECT * FROM users WHERE id IN (1);
```

这是正常的。

---

## 六、什么时候不要 Preload

不要在文章列表页一次 Preload：

```go
Preload("Comments.User")
```

如果每篇文章都有很多评论，列表会非常重。

应该拆成：

```text
文章列表接口
文章详情接口
评论分页接口
```

---

## 七、常见错误

### 1. 详情页用复杂 Joins 拼嵌套结构

如果目标是完整对象，`Preload` 通常更清晰。

### 2. 列表页 Preload 过多关联

列表页要轻，只加载必要字段。

### 3. Scan 字段别名不匹配

```sql
u.username AS author_name
```

对应：

```go
AuthorName string
```

---

## 八、本节达标标准

学完本节后，你应该能够做到：

- 解释 `Preload` 和 `Joins` 的差异。
- 用 `Preload` 查询文章详情。
- 用 `Joins` 查询文章列表 DTO。
- 按关联表字段过滤。
- 避免列表页加载过多关联。

---

## 颗粒度补强：本节应该如何真正练会

这一节不要只停留在“看懂概念”。建议你按下面的方式把「Joins与Preload查询取舍」练成可以在项目中使用的能力。

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
