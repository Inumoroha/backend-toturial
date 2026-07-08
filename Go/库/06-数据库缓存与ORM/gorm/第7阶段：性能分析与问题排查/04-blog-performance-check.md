# 04 博客系统性能体检

本节目标：像检查真实项目一样，对博客系统做一次 GORM 性能体检。你会打开 SQL 日志，检查文章列表、文章详情、热门文章、分类统计，并给出优化方案。

---

## 一、体检对象

重点检查这些接口：

```text
GET /api/articles
GET /api/articles/:id
GET /api/articles/hot
GET /api/categories/stats
```

它们分别代表：

- 分页列表
- 关联详情
- 排序排行
- 聚合统计

---

## 二、开启 SQL 日志

开发环境：

```go
db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
    Logger: logger.Default.LogMode(logger.Info),
})
```

如果你不想全局开启，可以对单个查询：

```go
db.Debug().Find(&articles)
```

---

## 三、检查文章列表

理想的文章列表 SQL 应该类似：

```sql
SELECT
    a.id,
    a.title,
    a.summary,
    a.status,
    a.created_at,
    u.username AS author_name,
    c.name AS category_name
FROM articles AS a
JOIN users AS u ON u.id = a.user_id
JOIN categories AS c ON c.id = a.category_id
WHERE a.deleted_at IS NULL
  AND a.status = 'published'
ORDER BY a.created_at DESC
LIMIT 10 OFFSET 0;
```

检查点：

- 是否有 `LIMIT`。
- 是否只查询列表需要的字段。
- 是否没有查询 `content`。
- 是否过滤了软删除数据。
- 排序字段是否有索引。

---

## 四、文章列表索引建议

如果常见查询是：

```text
status = published
category_id = ?
order by created_at desc
```

可以考虑：

```go
CategoryID uint      `gorm:"index:idx_article_category_status_created"`
Status     string    `gorm:"index:idx_article_category_status_created"`
CreatedAt  time.Time `gorm:"index:idx_article_category_status_created"`
```

迁移后检查：

```sql
SHOW INDEX FROM articles;
```

是否真的需要联合索引，要看数据量和查询频率。学习阶段先理解设计原因。

---

## 五、检查文章详情

文章详情常见写法：

```go
db.Preload("User").
    Preload("Category").
    Preload("Tags").
    Preload("Comments.User").
    First(&article, id)
```

检查点：

- SQL 条数是否稳定。
- 是否出现按评论数量循环查询用户。
- 评论是否需要分页。
- 是否加载了不需要的关联。

如果文章评论可能很多，不建议详情页一次加载所有评论。可以拆成：

```text
GET /api/articles/:id
GET /api/articles/:id/comments?page=1&page_size=20
```

这是接口设计层面的性能优化。

---

## 六、检查 N+1

如果日志出现类似：

```sql
SELECT * FROM users WHERE id = 1;
SELECT * FROM users WHERE id = 2;
SELECT * FROM users WHERE id = 3;
...
```

并且这些 SQL 是在循环中产生的，就要警惕 N+1。

修复方式：

```go
db.Preload("User").Find(&articles)
```

或者针对扁平列表直接使用 `JOIN`。

---

## 七、检查热门文章

热门文章：

```go
db.Where("status = ?", "published").
    Order("view_count DESC").
    Limit(10).
    Find(&articles)
```

如果数据量大，可以考虑：

- 给 `status` 加索引。
- 给 `view_count` 加索引。
- 使用缓存保存热门榜单。
- 定时计算热门文章，而不是每次实时排序。

学习阶段先做到：知道这类排序查询可能变慢。

---

## 八、检查分类统计

统计每个分类文章数：

```sql
SELECT c.id, c.name, COUNT(a.id) AS total
FROM categories c
LEFT JOIN articles a ON a.category_id = c.id AND a.deleted_at IS NULL
GROUP BY c.id, c.name;
```

检查点：

- `articles.category_id` 是否有索引。
- 数据量大时是否需要缓存统计结果。
- 是否需要只统计已发布文章。

如果业务要求只统计已发布文章，条件要写完整：

```sql
AND a.status = 'published'
```

---

## 九、性能体检清单

对每个接口记录：

```markdown
## 接口

## 生成 SQL

## SQL 条数

## 是否分页

## 是否存在 N+1

## 是否查询了多余字段

## 可能需要的索引

## 优化方案
```

---

## 十、常见错误

### 1. 看到 Preload 多条 SQL 就以为慢

多条 SQL 不一定慢。N+1 的问题是 SQL 条数随着数据量线性增长。

### 2. 只加索引不看查询

索引要服务于查询模式。先看 SQL，再设计索引。

### 3. 所有统计都实时算

数据量大后，热门文章、分类计数、阅读榜单可能需要缓存或预聚合。

### 4. 列表页加载评论

评论一般属于详情页或单独分页接口，不适合文章列表页一次加载。

---

## 十一、本节达标标准

学完本节后，你应该能够做到：

- 打开 GORM SQL 日志。
- 记录一个接口的 SQL 条数。
- 判断是否存在 N+1。
- 判断列表页是否查询了多余字段。
- 为文章列表设计合理索引。
- 对热门文章和统计接口提出缓存思路。
