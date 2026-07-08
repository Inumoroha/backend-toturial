# 03 原生 SQL 与结果扫描

本节目标：围绕「raw sql scan」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

GORM 的链式 API 适合大多数 CRUD 和简单查询，但复杂统计、复杂联表、数据库特定语法，有时直接写 SQL 更清楚。

会用 GORM 不等于不能写 SQL。优秀的后端工程师应该两者都能用，并知道何时取舍。

## 1. Raw 查询

```go
type ArticleSummary struct {
    ID       uint
    Title    string
    Username string
}

var results []ArticleSummary

err := db.Raw(`
    SELECT a.id, a.title, u.username
    FROM articles a
    JOIN users u ON u.id = a.user_id
    WHERE a.status = ?
    ORDER BY a.created_at DESC
    LIMIT ?
`, "published", 10).Scan(&results).Error
```

重点：

- 参数仍然使用 `?` 占位符。
- 不要拼接用户输入。
- `Scan` 的结构体字段要能匹配查询列。

## 2. Exec 执行 SQL

`Exec` 用于不需要返回行数据的 SQL：

```go
err := db.Exec(`
    UPDATE articles
    SET view_count = view_count + 1
    WHERE id = ?
`, articleID).Error
```

适合：

- 批量更新
- 执行维护 SQL
- 调用某些数据库特定能力

## 3. Scan 到自定义结构体

```go
type CategoryStats struct {
    CategoryID   uint
    CategoryName string
    ArticleTotal int64
}

var stats []CategoryStats

err := db.Raw(`
    SELECT c.id AS category_id,
           c.name AS category_name,
           COUNT(a.id) AS article_total
    FROM categories c
    LEFT JOIN articles a ON a.category_id = c.id
    GROUP BY c.id, c.name
`).Scan(&stats).Error
```

SQL 里的别名要和结构体字段对应：

- `category_id` -> `CategoryID`
- `category_name` -> `CategoryName`
- `article_total` -> `ArticleTotal`

## 4. Pluck 获取单列

```go
var emails []string

err := db.Model(&User{}).
    Where("deleted_at IS NULL").
    Pluck("email", &emails).Error
```

适合只取一列，比如 ID 列表、邮箱列表。

## 5. Row 和 Rows

取单行：

```go
row := db.Model(&Article{}).
    Select("COUNT(*)").
    Where("status = ?", "published").
    Row()

var total int64
if err := row.Scan(&total); err != nil {
    return err
}
```

遍历多行：

```go
rows, err := db.Model(&Article{}).
    Select("id, title").
    Rows()
if err != nil {
    return err
}
defer rows.Close()

for rows.Next() {
    var id uint
    var title string
    if err := rows.Scan(&id, &title); err != nil {
        return err
    }
}
```

大多数业务用 `Find` 和 `Scan` 就够了，`Rows` 更偏底层。

## 6. 什么时候用原生 SQL

适合使用原生 SQL：

- 查询逻辑复杂，GORM 链式写法反而难读。
- 使用窗口函数、CTE 等数据库高级语法。
- 复杂报表统计。
- 需要手动优化 SQL。
- 团队已经沉淀了可维护 SQL。

不适合滥用原生 SQL：

- 简单 CRUD。
- 简单条件查询。
- 普通分页。
- 简单关联预加载。

## 7. 本节练习

- [ ] 用 `Raw` 查询文章标题和作者用户名。
- [ ] 用 `Exec` 给文章阅读量加 1。
- [ ] 用 `Raw` 统计每个分类文章数量。
- [ ] 用 `Pluck` 查询所有用户邮箱。
- [ ] 用 `Rows` 遍历文章 ID 和标题。

## 小结

GORM 和 SQL 不是对立关系。

```text
简单业务：优先 GORM API
复杂统计：优先清晰 SQL
性能瓶颈：看执行计划和真实 SQL
```
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
