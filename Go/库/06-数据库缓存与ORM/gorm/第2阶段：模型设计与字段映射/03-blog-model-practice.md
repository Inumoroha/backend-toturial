# 03 博客系统模型设计练习

本节目标：围绕「blog model practice」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

本节开始设计一个博客系统。这个系统会贯穿后面的查询、关联、工程化和项目实战阶段。

## 1. 业务需求

博客系统需要支持：

- 用户注册和登录。
- 用户发布文章。
- 文章属于一个分类。
- 文章可以有多个标签。
- 用户可以评论文章。
- 评论支持软删除。

## 2. 表关系草图

```text
users 1 --- n articles
users 1 --- n comments
categories 1 --- n articles
articles 1 --- n comments
articles n --- n tags
```

多对多关系需要中间表：

```text
article_tags
```

## 3. User 模型

```go
type User struct {
    ID           uint           `gorm:"primaryKey"`
    Username     string         `gorm:"size:64;not null;uniqueIndex"`
    Email        string         `gorm:"size:128;not null;uniqueIndex"`
    PasswordHash string         `gorm:"size:255;not null"`
    CreatedAt    time.Time
    UpdatedAt    time.Time
    DeletedAt    gorm.DeletedAt `gorm:"index"`
}
```

说明：

- 不要存明文密码，存 `PasswordHash`。
- `Username` 和 `Email` 都应该唯一。
- 用户是否允许软删除取决于业务，这里保留软删除，便于后台管理。

## 4. Category 模型

```go
type Category struct {
    ID        uint   `gorm:"primaryKey"`
    Name      string `gorm:"size:64;not null;uniqueIndex"`
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

分类通常不频繁删除，可以先不加软删除。

## 5. Tag 模型

```go
type Tag struct {
    ID        uint   `gorm:"primaryKey"`
    Name      string `gorm:"size:64;not null;uniqueIndex"`
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

标签和文章是多对多关系，后续关联阶段会补充 `many2many`。

## 6. Article 模型

```go
type Article struct {
    ID         uint           `gorm:"primaryKey"`
    UserID     uint           `gorm:"not null;index"`
    CategoryID uint           `gorm:"not null;index"`
    Title      string         `gorm:"size:200;not null"`
    Summary    string         `gorm:"size:500"`
    Content    string         `gorm:"type:text;not null"`
    Status     string         `gorm:"size:32;not null;index;default:draft"`
    ViewCount  int            `gorm:"not null;default:0"`
    CreatedAt  time.Time
    UpdatedAt  time.Time
    DeletedAt  gorm.DeletedAt `gorm:"index"`
}
```

字段说明：

- `UserID`：作者 ID。
- `CategoryID`：分类 ID。
- `Status`：文章状态，例如 `draft`、`published`。
- `ViewCount`：阅读量。
- `DeletedAt`：文章通常需要软删除。

## 7. Comment 模型

```go
type Comment struct {
    ID        uint           `gorm:"primaryKey"`
    ArticleID uint           `gorm:"not null;index"`
    UserID    uint           `gorm:"not null;index"`
    Content   string         `gorm:"type:text;not null"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

评论建议软删除，因为删除评论后仍然可能要做后台审计。

## 8. AutoMigrate

```go
err := db.AutoMigrate(
    &User{},
    &Category{},
    &Tag{},
    &Article{},
    &Comment{},
)
if err != nil {
    log.Fatal(err)
}
```

## 9. 为什么现在不写关联字段

你可能会问：为什么 `Article` 里没有写 `User User`、`Tags []Tag`？

因为本阶段先关注表字段和约束。关联字段会在 `04-associations` 阶段系统补上。先把单表设计清楚，再学关联，会更稳。

## 10. 本节练习

- [ ] 把 5 个模型写到 Go 项目里。
- [ ] 执行 `AutoMigrate`。
- [ ] 在数据库客户端查看表结构。
- [ ] 确认唯一索引是否创建成功。
- [ ] 确认普通索引是否创建成功。
- [ ] 插入 1 个用户、1 个分类、1 篇文章、1 条评论。

## 11. 设计复盘问题

写完模型后，回答：

- 哪些字段必须唯一？
- 哪些字段适合加索引？
- 哪些表需要软删除？
- 为什么文章正文用 `text`？
- 为什么密码字段叫 `PasswordHash`？
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
