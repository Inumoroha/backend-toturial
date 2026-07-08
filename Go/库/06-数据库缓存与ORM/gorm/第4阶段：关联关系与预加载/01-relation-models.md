# 01 关系类型与模型定义

本节目标：围绕「relation models」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

数据库表之间通常不是孤立的。文章需要作者，评论需要文章，订单需要订单明细。这些关系在 GORM 中通过结构体字段和外键表达。

## 1. Belongs To：文章属于用户

```go
type Article struct {
    ID     uint
    UserID uint `gorm:"not null;index"`
    User   User
    Title  string
}
```

含义：

- `Article` 表有 `user_id` 字段。
- `Article` 属于一个 `User`。
- GORM 默认使用 `UserID` 作为外键。

查询文章时可以加载作者：

```go
db.Preload("User").First(&article, id)
```

## 2. Has Many：用户拥有多篇文章

```go
type User struct {
    ID       uint
    Username string
    Articles []Article
}
```

含义：

- 一个用户可以有多篇文章。
- `articles.user_id` 指向 `users.id`。

查询用户和文章：

```go
db.Preload("Articles").First(&user, userID)
```

## 3. 文章属于分类

```go
type Category struct {
    ID       uint
    Name     string
    Articles []Article
}

type Article struct {
    ID         uint
    UserID     uint
    CategoryID uint `gorm:"not null;index"`
    Category   Category
    Title      string
}
```

## 4. 文章拥有多条评论

```go
type Article struct {
    ID       uint
    Title    string
    Comments []Comment
}

type Comment struct {
    ID        uint
    ArticleID uint `gorm:"not null;index"`
    UserID    uint `gorm:"not null;index"`
    User      User
    Content   string
}
```

评论既属于文章，也属于用户。

## 5. Many To Many：文章和标签

一篇文章可以有多个标签，一个标签也可以属于多篇文章。

```go
type Article struct {
    ID    uint
    Title string
    Tags  []Tag `gorm:"many2many:article_tags;"`
}

type Tag struct {
    ID       uint
    Name     string
    Articles []Article `gorm:"many2many:article_tags;"`
}
```

GORM 会使用中间表 `article_tags`。

中间表通常包含：

```text
article_id
tag_id
```

## 6. 完整博客关联模型

```go
type User struct {
    ID       uint
    Username string
    Email    string
    Articles []Article
    Comments []Comment
}

type Category struct {
    ID       uint
    Name     string
    Articles []Article
}

type Article struct {
    ID         uint
    UserID     uint
    User       User
    CategoryID uint
    Category   Category
    Title      string
    Content    string
    Tags       []Tag `gorm:"many2many:article_tags;"`
    Comments   []Comment
}

type Tag struct {
    ID       uint
    Name     string
    Articles []Article `gorm:"many2many:article_tags;"`
}

type Comment struct {
    ID        uint
    ArticleID uint
    Article   Article
    UserID    uint
    User      User
    Content   string
}
```

真实项目里还要加上字段标签、时间字段和索引，这里先突出关联关系。

## 7. 外键命名约定

GORM 默认使用：

```text
关联结构体名 + ID
```

例如：

- `User` -> `UserID`
- `Category` -> `CategoryID`
- `Article` -> `ArticleID`

如果你按默认约定命名，配置会少很多。

## 本节练习

- [ ] 给 `Article` 添加 `User` 关联。
- [ ] 给 `User` 添加 `Articles` 关联。
- [ ] 给 `Article` 添加 `Category` 关联。
- [ ] 给 `Article` 添加 `Comments` 关联。
- [ ] 给 `Comment` 添加 `User` 关联。
- [ ] 给 `Article` 和 `Tag` 添加多对多关联。
- [ ] 执行 `AutoMigrate`，观察是否生成 `article_tags` 表。
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
