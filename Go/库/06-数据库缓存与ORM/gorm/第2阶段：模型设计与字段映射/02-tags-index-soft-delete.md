# 02 字段标签、索引与软删除

本节目标：围绕「tags index soft delete」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

字段标签决定了结构体字段如何映射到数据库列。GORM 的 tag 功能很强，但不要滥用。好的模型应该让业务含义清晰，让数据库约束足够明确。

## 1. 常用字段标签

```go
type User struct {
    ID       uint   `gorm:"primaryKey"`
    Username string `gorm:"size:64;not null;uniqueIndex"`
    Email    string `gorm:"size:128;not null;uniqueIndex"`
    Age      int    `gorm:"default:18"`
}
```

含义：

- `primaryKey`：主键。
- `size:64`：字符串长度。
- `not null`：不能为空。
- `uniqueIndex`：唯一索引。
- `default:18`：默认值。

## 2. 字段类型

你可以指定数据库类型：

```go
type Article struct {
    ID      uint
    Title   string `gorm:"type:varchar(200);not null"`
    Content string `gorm:"type:text"`
}
```

建议：

- 标题、用户名、邮箱等有限长度字符串，用 `size` 或 `varchar`。
- 正文、描述等长文本，用 `text`。
- 金额不要用 `float64`，真实项目里建议用整数分，或 decimal 类型。

## 3. 普通索引

```go
type Article struct {
    ID        uint
    UserID    uint      `gorm:"index"`
    CreatedAt time.Time `gorm:"index"`
}
```

适合建立索引的字段：

- 外键字段，例如 `user_id`
- 高频查询字段，例如 `status`
- 高频排序字段，例如 `created_at`

## 4. 唯一索引

```go
type User struct {
    ID    uint
    Email string `gorm:"size:128;not null;uniqueIndex"`
}
```

唯一索引用于保证数据不重复。比如邮箱、用户名、手机号。

注意：唯一索引不是前端校验的替代品，而是数据库层面的最后防线。

## 5. 联合索引

```go
type Article struct {
    ID         uint
    CategoryID uint   `gorm:"index:idx_category_status"`
    Status     string `gorm:"index:idx_category_status"`
}
```

这会创建一个联合索引，适合类似下面的查询：

```sql
SELECT * FROM articles WHERE category_id = 1 AND status = 'published';
```

联合索引要根据真实查询设计，不要凭感觉乱加。

## 6. 软删除

模型中包含 `gorm.DeletedAt` 时，GORM 默认执行软删除：

```go
type Comment struct {
    ID        uint
    Content   string
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

删除时：

```go
db.Delete(&comment)
```

实际执行的不是 `DELETE`，而是类似：

```sql
UPDATE comments SET deleted_at = '...' WHERE id = 1;
```

普通查询会自动排除已软删除的数据：

```sql
WHERE deleted_at IS NULL
```

## 7. 查询软删除数据

包括已删除数据：

```go
db.Unscoped().Find(&comments)
```

永久删除：

```go
db.Unscoped().Delete(&comment)
```

永久删除要谨慎，通常只在后台清理任务或测试数据中使用。

## 8. 零值更新问题

使用结构体更新时，GORM 默认不会更新零值字段：

```go
db.Model(&user).Updates(User{
    Name: "",
    Age:  0,
})
```

这里 `Name` 和 `Age` 可能不会被更新，因为空字符串和 0 是 Go 的零值。

如果确实要更新零值，可以用 `map`：

```go
db.Model(&user).Updates(map[string]any{
    "name": "",
    "age":  0,
})
```

或者显式选择字段：

```go
db.Model(&user).Select("name", "age").Updates(User{
    Name: "",
    Age:  0,
})
```

这是 GORM 新手最常见的坑之一。

## 本节练习

- [ ] 给 `users.email` 添加唯一索引。
- [ ] 给 `articles.user_id` 添加普通索引。
- [ ] 给 `articles.category_id` 和 `articles.status` 添加联合索引。
- [ ] 给 `comments` 添加软删除字段。
- [ ] 测试软删除后普通查询是否还能查到数据。
- [ ] 测试结构体更新零值和 `map` 更新零值的区别。
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
