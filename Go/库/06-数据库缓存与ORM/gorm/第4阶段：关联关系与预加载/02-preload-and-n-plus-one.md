# 02 预加载与 N+1 查询

本节目标：围绕「preload and n plus one」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

关联关系定义好以后，下一步是查询关联数据。GORM 里最常用的是 `Preload`。

## 1. 查询文章时加载作者

```go
var article Article

err := db.Preload("User").
    First(&article, articleID).Error
```

GORM 通常会执行两条 SQL：

```sql
SELECT * FROM articles WHERE id = 1 LIMIT 1;
SELECT * FROM users WHERE id IN (1);
```

## 2. 查询文章时加载分类、标签、评论

```go
var article Article

err := db.Preload("User").
    Preload("Category").
    Preload("Tags").
    Preload("Comments").
    First(&article, articleID).Error
```

这样文章详情页就可以展示：

- 标题
- 作者
- 分类
- 标签
- 评论

## 3. 嵌套预加载

评论还需要显示评论人：

```go
err := db.Preload("User").
    Preload("Category").
    Preload("Tags").
    Preload("Comments.User").
    First(&article, articleID).Error
```

`Comments.User` 表示先加载评论，再加载评论对应的用户。

## 4. 条件预加载

只加载未删除或已通过审核的评论：

```go
err := db.Preload("Comments", "status = ?", "approved").
    First(&article, articleID).Error
```

也可以给预加载排序：

```go
err := db.Preload("Comments", func(db *gorm.DB) *gorm.DB {
    return db.Order("created_at DESC")
}).First(&article, articleID).Error
```

## 5. 查询用户和文章列表

```go
var user User

err := db.Preload("Articles", func(db *gorm.DB) *gorm.DB {
    return db.Where("status = ?", "published").
        Order("created_at DESC")
}).First(&user, userID).Error
```

这适合用户主页。

## 6. 什么是 N+1 查询

错误示例：

```go
var articles []Article
db.Find(&articles)

for i := range articles {
    db.First(&articles[i].User, articles[i].UserID)
}
```

如果有 100 篇文章：

- 1 条 SQL 查询文章列表。
- 100 条 SQL 查询作者。

这就是 N+1 查询。

## 7. 用 Preload 避免 N+1

正确写法：

```go
var articles []Article

err := db.Preload("User").
    Where("status = ?", "published").
    Limit(100).
    Find(&articles).Error
```

GORM 会用类似 `IN` 的查询一次加载所有作者：

```sql
SELECT * FROM users WHERE id IN (...);
```

## 8. Preload 和 Joins 的区别

简单理解：

- `Preload`：用额外 SQL 查询关联数据，再组装到结构体。
- `Joins`：使用 SQL JOIN，把数据在一条 SQL 里查出来。

大多数关联对象加载，优先用 `Preload`。

适合 `Joins` 的场景：

- 按关联表字段过滤。
- 按关联表字段排序。
- 查询结果本身就是扁平 DTO。

## 本节练习

- [ ] 查询文章详情，加载作者和分类。
- [ ] 查询文章详情，加载标签。
- [ ] 查询文章详情，加载评论和评论人。
- [ ] 查询用户主页，加载用户已发布文章。
- [ ] 故意写一个 N+1 查询，然后用 `Preload` 优化。
- [ ] 打印 SQL，对比优化前后的 SQL 条数。

## 常见坑

- 关联字段名写错，例如 `Preload("users")`。
- 忘记定义结构体关联字段。
- 列表页预加载过多无用关联。
- 详情页没有预加载必要关联。
- 没观察 SQL 条数，不知道自己写出了 N+1。
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
