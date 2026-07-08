# 04 博客查询 Repository 实战

本节目标：把查询能力放进一个接近真实项目的 Repository 中。你会实现文章列表、文章详情、关键词搜索、分类统计和热门文章查询，并理解每个查询背后的 SQL。

---

## 一、本节要实现的查询

博客系统常见查询：

```text
1. 文章分页列表
2. 按分类筛选文章
3. 按关键词搜索文章
4. 文章详情
5. 统计分类文章数
6. 查询热门文章
```

这些查询足够覆盖大部分后台管理和内容系统的基本场景。

---

## 二、推荐文件位置

```text
internal
└── repository
    └── article_repository.go
```

Repository 只负责数据库读写，不处理 HTTP 请求，也不直接返回 Gin 的响应。

---

## 三、定义查询参数

```go
package repository

type ArticleListQuery struct {
    Page       int
    PageSize   int
    CategoryID uint
    Keyword    string
    Status     string
}
```

为什么用结构体？

因为列表查询条件会越来越多。如果每个条件都作为函数参数，函数签名会变得很长。

---

## 四、定义返回结构

列表页通常不返回正文：

```go
type ArticleListItem struct {
    ID           uint
    Title        string
    Summary      string
    Status       string
    ViewCount    int
    AuthorName   string
    CategoryName string
    CreatedAt    time.Time
}
```

这里是扁平结构，适合列表页直接返回给前端。

需要导入：

```go
import "time"
```

---

## 五、Repository 结构

```go
type ArticleRepository struct {
    db *gorm.DB
}

func NewArticleRepository(db *gorm.DB) *ArticleRepository {
    return &ArticleRepository{db: db}
}
```

需要导入：

```go
import "gorm.io/gorm"
```

---

## 六、分页参数标准化

```go
func normalizePagination(page, pageSize int) (int, int) {
    if page < 1 {
        page = 1
    }
    if pageSize < 1 {
        pageSize = 10
    }
    if pageSize > 100 {
        pageSize = 100
    }
    return page, pageSize
}
```

为什么限制 `pageSize`？

如果不限制，用户可以传 `page_size=100000`，一次查出大量数据，拖慢数据库和接口。

---

## 七、实现文章列表

```go
func (r *ArticleRepository) List(ctx context.Context, q ArticleListQuery) ([]ArticleListItem, int64, error) {
    page, pageSize := normalizePagination(q.Page, q.PageSize)
    offset := (page - 1) * pageSize

    base := r.db.WithContext(ctx).
        Table("articles AS a").
        Joins("JOIN users AS u ON u.id = a.user_id").
        Joins("JOIN categories AS c ON c.id = a.category_id").
        Where("a.deleted_at IS NULL")

    if q.CategoryID > 0 {
        base = base.Where("a.category_id = ?", q.CategoryID)
    }

    if q.Keyword != "" {
        base = base.Where("a.title LIKE ?", "%"+q.Keyword+"%")
    }

    if q.Status != "" {
        base = base.Where("a.status = ?", q.Status)
    }

    var total int64
    if err := base.Count(&total).Error; err != nil {
        return nil, 0, err
    }

    var items []ArticleListItem
    err := base.Select(`
            a.id,
            a.title,
            a.summary,
            a.status,
            a.view_count,
            u.username AS author_name,
            c.name AS category_name,
            a.created_at
        `).
        Order("a.created_at DESC").
        Limit(pageSize).
        Offset(offset).
        Scan(&items).Error
    if err != nil {
        return nil, 0, err
    }

    return items, total, nil
}
```

需要导入：

```go
import "context"
```

---

## 八、这段查询背后的 SQL

大致是：

```sql
SELECT
    a.id,
    a.title,
    a.summary,
    a.status,
    a.view_count,
    u.username AS author_name,
    c.name AS category_name,
    a.created_at
FROM articles AS a
JOIN users AS u ON u.id = a.user_id
JOIN categories AS c ON c.id = a.category_id
WHERE a.deleted_at IS NULL
  AND a.category_id = ?
  AND a.title LIKE ?
  AND a.status = ?
ORDER BY a.created_at DESC
LIMIT ? OFFSET ?;
```

注意这里用 `Joins` 是因为列表页需要作者名和分类名，而且返回的是扁平 DTO。

如果你想返回嵌套结构体，也可以使用 `Preload`。

---

## 九、Count 查询的注意点

上面的代码复用了 `base` 查询：

```go
base.Count(&total)
```

再继续：

```go
base.Select(...).Limit(...).Scan(&items)
```

简单场景可以这样写。复杂场景里，如果你发现链式状态互相影响，可以把构造 base query 的逻辑抽成函数，每次重新创建。

---

## 十、实现文章详情

文章详情需要关联：

- 作者
- 分类
- 标签
- 评论
- 评论用户

如果你的 `model.Article` 定义了关联字段，可以写：

```go
func (r *ArticleRepository) FindDetail(ctx context.Context, id uint) (*model.Article, error) {
    var article model.Article

    err := r.db.WithContext(ctx).
        Preload("User").
        Preload("Category").
        Preload("Tags").
        Preload("Comments", func(db *gorm.DB) *gorm.DB {
            return db.Order("created_at ASC")
        }).
        Preload("Comments.User").
        First(&article, "id = ?", id).Error
    if err != nil {
        return nil, err
    }

    return &article, nil
}
```

详情页适合使用 `Preload`，因为你希望拿到结构化的文章对象。

---

## 十一、热门文章

```go
func (r *ArticleRepository) ListHot(ctx context.Context, limit int) ([]model.Article, error) {
    if limit <= 0 || limit > 50 {
        limit = 10
    }

    var articles []model.Article
    err := r.db.WithContext(ctx).
        Where("status = ?", "published").
        Order("view_count DESC").
        Order("created_at DESC").
        Limit(limit).
        Find(&articles).Error

    return articles, err
}
```

这个查询应该考虑索引：

```text
status
view_count
created_at
```

是否建立联合索引，要看真实查询频率和数据量。

---

## 十二、分类文章数统计

```go
type CategoryArticleCount struct {
    CategoryID   uint
    CategoryName string
    Total        int64
}

func (r *ArticleRepository) CountByCategory(ctx context.Context) ([]CategoryArticleCount, error) {
    var results []CategoryArticleCount

    err := r.db.WithContext(ctx).
        Table("categories AS c").
        Select("c.id AS category_id, c.name AS category_name, COUNT(a.id) AS total").
        Joins("LEFT JOIN articles AS a ON a.category_id = c.id AND a.deleted_at IS NULL").
        Group("c.id, c.name").
        Order("c.id ASC").
        Scan(&results).Error

    return results, err
}
```

这里使用 `LEFT JOIN`，是为了让没有文章的分类也能显示出来，数量为 0。

---

## 十三、常见错误

### 1. 列表页返回正文

列表页不要查询 `content`。正文应该在详情页查。

### 2. 搜索条件直接拼接

错误：

```go
Where("a.title LIKE '%" + q.Keyword + "%'")
```

正确：

```go
Where("a.title LIKE ?", "%"+q.Keyword+"%")
```

### 3. 忘记软删除过滤

使用 GORM Model 查询时，GORM 会自动处理软删除。但使用 `Table("articles AS a")` 时，你需要自己写：

```go
Where("a.deleted_at IS NULL")
```

### 4. 详情页循环查关联

不要先查文章，再循环查标签、评论、评论用户。用 `Preload`。

---

## 十四、本节达标标准

学完本节后，你应该能够做到：

- 写出文章列表 Repository。
- 支持分页、分类、关键词、状态筛选。
- 返回列表总数。
- 列表页只查询必要字段。
- 使用 `Joins` 返回扁平 DTO。
- 使用 `Preload` 查询文章详情。
- 写出分类文章数统计查询。
- 解释什么时候用 `Joins`，什么时候用 `Preload`。
