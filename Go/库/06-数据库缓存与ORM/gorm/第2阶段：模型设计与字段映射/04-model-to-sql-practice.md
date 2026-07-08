# 04 从业务模型到数据库表结构

本节目标：把“写结构体”提升到“设计表结构”的层次。你要能看到一个 GORM Model，就知道它大概会生成什么表；也要能看到业务需求，反推出合适的字段、索引、约束和软删除策略。

---

## 一、为什么模型设计不能随便写

GORM 让建表变简单，但这也带来一个问题：很多初学者会以为结构体字段写出来就完事了。

真实项目里，模型设计会影响：

- 查询是否方便。
- 数据是否能保持一致。
- 是否容易出现重复数据。
- 是否容易误删数据。
- 后续接口是否好写。
- 性能优化有没有空间。

所以设计模型时，要同时考虑 Go 结构体、数据库表、业务规则和查询模式。

---

## 二、从需求开始

以博客系统为例，先写需求，不要急着写代码：

```text
用户可以注册和登录。
用户可以发布文章。
文章属于一个分类。
文章可以绑定多个标签。
用户可以评论文章。
文章和评论支持软删除。
文章有草稿和已发布状态。
列表页需要按分类、状态、时间查询。
```

从这些需求中能推导出：

- 用户邮箱需要唯一。
- 文章需要 `user_id`。
- 文章需要 `category_id`。
- 文章需要 `status`。
- 评论需要 `article_id` 和 `user_id`。
- 标签和文章是多对多。
- `user_id`、`category_id`、`status`、`created_at` 经常参与查询，应该考虑索引。

---

## 三、先设计 User

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

### 字段解释

| 字段 | 含义 | 设计理由 |
| --- | --- | --- |
| `ID` | 主键 | 用于唯一标识用户 |
| `Username` | 用户名 | 展示和登录都可能用到，设置唯一 |
| `Email` | 邮箱 | 注册登录常用，必须唯一 |
| `PasswordHash` | 密码哈希 | 不能存明文密码 |
| `CreatedAt` | 创建时间 | GORM 自动维护 |
| `UpdatedAt` | 更新时间 | GORM 自动维护 |
| `DeletedAt` | 软删除时间 | 删除用户时保留历史数据 |

### 可能生成的 SQL 直觉

不同数据库生成的 SQL 会有差异，但大概会包含：

```sql
CREATE TABLE users (
    id bigint unsigned AUTO_INCREMENT,
    username varchar(64) NOT NULL,
    email varchar(128) NOT NULL,
    password_hash varchar(255) NOT NULL,
    created_at datetime,
    updated_at datetime,
    deleted_at datetime,
    PRIMARY KEY (id)
);
```

并创建唯一索引：

```sql
CREATE UNIQUE INDEX idx_users_username ON users(username);
CREATE UNIQUE INDEX idx_users_email ON users(email);
```

---

## 四、设计 Article

```go
type Article struct {
    ID         uint           `gorm:"primaryKey"`
    UserID     uint           `gorm:"not null;index"`
    CategoryID uint           `gorm:"not null;index:idx_articles_category_status_created"`
    Title      string         `gorm:"size:200;not null"`
    Summary    string         `gorm:"size:500"`
    Content    string         `gorm:"type:text;not null"`
    Status     string         `gorm:"size:32;not null;default:draft;index:idx_articles_category_status_created"`
    ViewCount  int            `gorm:"not null;default:0"`
    CreatedAt  time.Time      `gorm:"index:idx_articles_category_status_created"`
    UpdatedAt  time.Time
    DeletedAt  gorm.DeletedAt `gorm:"index"`
}
```

### 为什么需要 `Summary`

文章列表页通常只展示摘要，不需要返回完整正文。

如果没有摘要字段，你可能会在列表页查询 `content`，导致：

- 数据传输变大。
- 查询变慢。
- 前端拿到不需要的大字段。

### 为什么 `Content` 用 `text`

标题长度有限，正文长度不可控。正文应该使用 `text` 类型，而不是 `varchar(255)`。

### 为什么有联合索引

文章列表常见查询：

```sql
SELECT id, title, summary, created_at
FROM articles
WHERE category_id = 1 AND status = 'published'
ORDER BY created_at DESC
LIMIT 10;
```

所以这里设计：

```go
CategoryID uint      `gorm:"index:idx_articles_category_status_created"`
Status     string    `gorm:"index:idx_articles_category_status_created"`
CreatedAt  time.Time `gorm:"index:idx_articles_category_status_created"`
```

它表达的是：这个组合经常一起查询。

---

## 五、设计 Category 和 Tag

分类：

```go
type Category struct {
    ID        uint      `gorm:"primaryKey"`
    Name      string    `gorm:"size:64;not null;uniqueIndex"`
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

标签：

```go
type Tag struct {
    ID        uint      `gorm:"primaryKey"`
    Name      string    `gorm:"size:64;not null;uniqueIndex"`
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

为什么名称唯一？

因为不希望出现两个同名分类，也不希望同一个标签被重复创建多次。

---

## 六、设计 Comment

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

评论为什么需要软删除？

因为评论可能涉及审核和追溯。用户看不到已删除评论，但后台可能仍需要查看记录。

---

## 七、把模型放到正确文件

推荐文件结构：

```text
internal
└── model
    ├── user.go
    ├── article.go
    ├── category.go
    ├── tag.go
    └── comment.go
```

每个文件只放相关模型，后续维护更清晰。

例如 `internal/model/user.go`：

```go
package model

import (
    "time"

    "gorm.io/gorm"
)

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

---

## 八、迁移并检查表结构

迁移：

```go
err := db.AutoMigrate(
    &model.User{},
    &model.Category{},
    &model.Tag{},
    &model.Article{},
    &model.Comment{},
)
if err != nil {
    log.Fatal(err)
}
```

在 MySQL 中查看表：

```sql
SHOW TABLES;
```

查看文章表结构：

```sql
SHOW CREATE TABLE articles;
```

查看索引：

```sql
SHOW INDEX FROM articles;
```

你要确认：

- `users.email` 有唯一索引。
- `articles.user_id` 有索引。
- `articles.category_id/status/created_at` 有联合索引。
- `comments.deleted_at` 有索引。

---

## 九、常见设计错误

### 1. 明文保存密码

错误：

```go
Password string
```

应该保存：

```go
PasswordHash string
```

密码哈希可以使用 bcrypt 等算法生成。

### 2. 列表接口直接查询正文

文章正文可能很大，列表页应该查询摘要字段。

### 3. 所有字段都加索引

索引不是越多越好。写入时索引也需要维护，乱加索引会拖慢写入。

### 4. 该唯一的不唯一

邮箱、用户名、分类名、标签名通常需要唯一约束。只在业务代码里判断不够，数据库也要兜底。

### 5. 不区分删除和隐藏

软删除适合“数据不应该立刻物理消失”的场景。文章、评论、用户通常可以软删除；临时日志、缓存数据不一定需要。

---

## 十、本节达标标准

学完本节后，你应该能够做到：

- 根据业务需求推导模型字段。
- 解释每个字段为什么存在。
- 知道哪些字段应该唯一。
- 知道哪些字段应该加索引。
- 知道哪些表适合软删除。
- 能通过 `SHOW CREATE TABLE` 检查 GORM 生成的表结构。
- 能通过 `SHOW INDEX` 检查索引是否符合预期。
