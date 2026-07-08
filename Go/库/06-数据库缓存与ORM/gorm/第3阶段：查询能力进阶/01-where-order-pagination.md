# 01 条件查询、排序与分页

本节目标：围绕「where order pagination」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

真实后端接口里，最常见的查询不是“查全部”，而是“按条件查一页”。本节重点学习 `Where`、`Order`、`Limit`、`Offset`。

## 1. 基础条件查询

查询已发布文章：

```go
var articles []Article

err := db.Where("status = ?", "published").
    Find(&articles).Error
```

对应 SQL：

```sql
SELECT * FROM articles WHERE status = 'published';
```

重点：永远优先使用占位符，不要拼接用户输入。

## 2. 多条件查询

查询某个分类下已发布的文章：

```go
err := db.Where("category_id = ? AND status = ?", categoryID, "published").
    Find(&articles).Error
```

也可以链式写：

```go
err := db.Where("category_id = ?", categoryID).
    Where("status = ?", "published").
    Find(&articles).Error
```

链式写法更适合动态条件。

## 3. 动态查询条件

文章列表通常有可选筛选条件：

- 分类 ID 可选
- 关键词可选
- 状态可选

```go
query := db.Model(&Article{})

if categoryID > 0 {
    query = query.Where("category_id = ?", categoryID)
}

if keyword != "" {
    query = query.Where("title LIKE ?", "%"+keyword+"%")
}

if status != "" {
    query = query.Where("status = ?", status)
}

err := query.Find(&articles).Error
```

这种写法是 Repository 层里非常常见的模式。

## 4. 选择字段

如果列表页不需要正文内容，就不要查询 `content`。

```go
err := db.Model(&Article{}).
    Select("id", "title", "summary", "created_at").
    Where("status = ?", "published").
    Find(&articles).Error
```

这样可以减少数据传输，尤其是文章正文很长时。

## 5. 排序

按发布时间倒序：

```go
err := db.Where("status = ?", "published").
    Order("created_at DESC").
    Find(&articles).Error
```

多字段排序：

```go
err := db.Order("view_count DESC").
    Order("created_at DESC").
    Find(&articles).Error
```

注意：如果排序字段来自用户输入，一定要做白名单校验，不能直接拼进去。

## 6. 分页

分页参数：

```go
page := 1
pageSize := 10
offset := (page - 1) * pageSize
```

GORM 查询：

```go
err := db.Where("status = ?", "published").
    Order("created_at DESC").
    Limit(pageSize).
    Offset(offset).
    Find(&articles).Error
```

## 7. 分页参数保护

不要信任前端传来的分页参数：

```go
func normalizePage(page, pageSize int) (int, int) {
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

这样可以避免用户一次请求几万条数据拖垮服务。

## 8. 查询单条与多条

查询单条：

```go
var article Article
err := db.Where("id = ? AND status = ?", id, "published").
    First(&article).Error
```

查询多条：

```go
var articles []Article
err := db.Where("status = ?", "published").
    Find(&articles).Error
```

区别：

- `First` 查不到会返回 `gorm.ErrRecordNotFound`。
- `Find` 查不到通常返回空切片。

## 9. 本节练习

- [ ] 查询某个用户发布的文章。
- [ ] 查询某个分类下已发布文章。
- [ ] 按创建时间倒序查询文章。
- [ ] 按阅读量倒序查询文章。
- [ ] 实现标题关键词搜索。
- [ ] 实现分页参数保护。
- [ ] 列表页只查询必要字段，不查询正文。

## 常见坑

- 无条件 `Find` 查询全表。
- 用户输入直接拼进 `Order`。
- 分页参数不限制最大值。
- 列表页查询了大字段。
- 忘记处理 `ErrRecordNotFound`。
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
