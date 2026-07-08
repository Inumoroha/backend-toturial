# 02 统计、分组与聚合查询

本节目标：围绕「count group aggregate」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

业务系统里不只有列表和详情，还有大量统计需求。例如文章总数、每个分类的文章数、热门文章排行、用户发布数量等。

## 1. Count 统计总数

统计已发布文章数量：

```go
var total int64

err := db.Model(&Article{}).
    Where("status = ?", "published").
    Count(&total).Error
```

注意：`Count` 通常要配合 `Model` 使用。

## 2. 分页时同时查询总数

接口常见返回：

```json
{
  "list": [],
  "total": 100,
  "page": 1,
  "page_size": 10
}
```

实现方式：

```go
query := db.Model(&Article{}).Where("status = ?", "published")

var total int64
if err := query.Count(&total).Error; err != nil {
    return err
}

var articles []Article
err := query.Order("created_at DESC").
    Limit(pageSize).
    Offset(offset).
    Find(&articles).Error
```

注意：复杂场景里，`Count` 和列表查询可能需要拆成两个独立 query，避免链式状态互相影响。

## 3. Group 分组

统计每个分类下的文章数量：

```go
type CategoryArticleCount struct {
    CategoryID uint
    Total      int64
}

var results []CategoryArticleCount

err := db.Model(&Article{}).
    Select("category_id, COUNT(*) AS total").
    Where("status = ?", "published").
    Group("category_id").
    Scan(&results).Error
```

对应 SQL：

```sql
SELECT category_id, COUNT(*) AS total
FROM articles
WHERE status = 'published'
GROUP BY category_id;
```

## 4. Having

查询文章数大于 10 的分类：

```go
err := db.Model(&Article{}).
    Select("category_id, COUNT(*) AS total").
    Where("status = ?", "published").
    Group("category_id").
    Having("COUNT(*) > ?", 10).
    Scan(&results).Error
```

`WHERE` 过滤分组前的数据，`HAVING` 过滤分组后的结果。

## 5. 聚合函数

统计总阅读量：

```go
type ViewStats struct {
    TotalViews int64
    MaxViews   int64
    AvgViews   float64
}

var stats ViewStats

err := db.Model(&Article{}).
    Select("SUM(view_count) AS total_views, MAX(view_count) AS max_views, AVG(view_count) AS avg_views").
    Where("status = ?", "published").
    Scan(&stats).Error
```

## 6. 热门文章

查询阅读量最高的前 10 篇文章：

```go
var articles []Article

err := db.Where("status = ?", "published").
    Order("view_count DESC").
    Limit(10).
    Find(&articles).Error
```

如果阅读量相同，可以追加创建时间排序：

```go
Order("view_count DESC, created_at DESC")
```

## 7. Distinct 去重

查询发布过文章的用户 ID：

```go
var userIDs []uint

err := db.Model(&Article{}).
    Distinct("user_id").
    Pluck("user_id", &userIDs).Error
```

`Pluck` 适合只取一列数据。

## 8. 本节练习

- [ ] 统计已发布文章总数。
- [ ] 实现文章分页列表，并返回总数。
- [ ] 统计每个分类的文章数量。
- [ ] 查询文章数大于 5 的分类。
- [ ] 统计所有文章总阅读量。
- [ ] 查询阅读量最高的前 10 篇文章。
- [ ] 查询发布过文章的用户 ID 列表。

## 常见坑

- 忘记 `Model(&Article{})` 导致 Count 不明确。
- 聚合查询结果字段名和结构体字段对不上。
- `WHERE` 和 `HAVING` 混用。
- 分页列表查询和 Count 查询互相污染链式条件。
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
