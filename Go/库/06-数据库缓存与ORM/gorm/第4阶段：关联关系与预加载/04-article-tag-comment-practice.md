# 04 文章、标签、评论关联实战

本节目标：完成博客系统中最常见的关联场景：创建文章并绑定标签、查询文章详情并加载作者/分类/标签/评论、编辑文章并替换标签、避免 N+1 查询。

---

## 一、本节业务场景

文章详情页通常需要展示：

```text
文章标题
文章正文
作者
分类
标签列表
评论列表
评论人的用户名
```

这背后涉及：

- `Article belongs to User`
- `Article belongs to Category`
- `Article many to many Tag`
- `Article has many Comment`
- `Comment belongs to User`

---

## 二、完整关联模型

建议放在：

```text
internal/model
```

### User

```go
type User struct {
    ID           uint           `gorm:"primaryKey"`
    Username     string         `gorm:"size:64;not null;uniqueIndex"`
    Email        string         `gorm:"size:128;not null;uniqueIndex"`
    PasswordHash string         `gorm:"size:255;not null"`
    Articles     []Article
    Comments     []Comment
    CreatedAt    time.Time
    UpdatedAt    time.Time
    DeletedAt    gorm.DeletedAt `gorm:"index"`
}
```

### Category

```go
type Category struct {
    ID        uint      `gorm:"primaryKey"`
    Name      string    `gorm:"size:64;not null;uniqueIndex"`
    Articles  []Article
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

### Tag

```go
type Tag struct {
    ID        uint      `gorm:"primaryKey"`
    Name      string    `gorm:"size:64;not null;uniqueIndex"`
    Articles  []Article `gorm:"many2many:article_tags;"`
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

### Article

```go
type Article struct {
    ID         uint           `gorm:"primaryKey"`
    UserID     uint           `gorm:"not null;index"`
    User       User
    CategoryID uint           `gorm:"not null;index"`
    Category   Category
    Title      string         `gorm:"size:200;not null"`
    Summary    string         `gorm:"size:500"`
    Content    string         `gorm:"type:text;not null"`
    Status     string         `gorm:"size:32;not null;default:draft;index"`
    Tags       []Tag          `gorm:"many2many:article_tags;"`
    Comments   []Comment
    CreatedAt  time.Time
    UpdatedAt  time.Time
    DeletedAt  gorm.DeletedAt `gorm:"index"`
}
```

### Comment

```go
type Comment struct {
    ID        uint           `gorm:"primaryKey"`
    ArticleID uint           `gorm:"not null;index"`
    Article   Article
    UserID    uint           `gorm:"not null;index"`
    User      User
    Content   string         `gorm:"type:text;not null"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

---

## 三、迁移后检查中间表

执行：

```go
db.AutoMigrate(
    &model.User{},
    &model.Category{},
    &model.Tag{},
    &model.Article{},
    &model.Comment{},
)
```

在数据库中检查：

```sql
SHOW TABLES;
```

你应该能看到：

```text
article_tags
```

再查看中间表：

```sql
SHOW CREATE TABLE article_tags;
```

它至少应该有：

```text
article_id
tag_id
```

这张表不是你手写的，而是 GORM 根据 `many2many:article_tags` 创建的。

---

## 四、创建文章并绑定标签

业务流程：

```text
1. 接收标题、正文、分类 ID、标签 ID 列表
2. 查询标签是否存在
3. 创建文章
4. 写入 article_tags 中间表
```

代码：

```go
type CreateArticleInput struct {
    UserID     uint
    CategoryID uint
    Title      string
    Summary    string
    Content    string
    TagIDs     []uint
}

func (r *ArticleRepository) CreateWithTags(ctx context.Context, input CreateArticleInput) (*model.Article, error) {
    var tags []model.Tag
    if len(input.TagIDs) > 0 {
        if err := r.db.WithContext(ctx).
            Where("id IN ?", input.TagIDs).
            Find(&tags).Error; err != nil {
            return nil, err
        }

        if len(tags) != len(input.TagIDs) {
            return nil, fmt.Errorf("some tags not found")
        }
    }

    article := &model.Article{
        UserID:     input.UserID,
        CategoryID: input.CategoryID,
        Title:      input.Title,
        Summary:    input.Summary,
        Content:    input.Content,
        Status:     "draft",
        Tags:       tags,
    }

    if err := r.db.WithContext(ctx).Create(article).Error; err != nil {
        return nil, err
    }

    return article, nil
}
```

注意：这个版本还没有显式事务。更完整的业务会在 Service 层使用事务。

---

## 五、查询文章详情

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

这里会发出多条 SQL，但不是 N+1。GORM 会按关联批量加载。

---

## 六、编辑文章并替换标签

编辑文章通常不是追加标签，而是替换成前端提交的新标签列表。

业务流程：

```text
1. 查询文章是否存在
2. 校验当前用户是否是作者
3. 更新文章标题、摘要、正文、分类
4. 查询新标签列表
5. Replace 标签关联
6. 所有步骤放进事务
```

代码示例：

```go
func (s *ArticleService) UpdateArticle(ctx context.Context, articleID uint, input UpdateArticleInput) error {
    return s.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        var article model.Article
        if err := tx.First(&article, articleID).Error; err != nil {
            return err
        }

        if article.UserID != input.UserID {
            return ErrForbidden
        }

        updates := map[string]any{
            "title":       input.Title,
            "summary":     input.Summary,
            "content":     input.Content,
            "category_id": input.CategoryID,
        }

        if err := tx.Model(&article).Updates(updates).Error; err != nil {
            return err
        }

        var tags []model.Tag
        if len(input.TagIDs) > 0 {
            if err := tx.Where("id IN ?", input.TagIDs).Find(&tags).Error; err != nil {
                return err
            }
            if len(tags) != len(input.TagIDs) {
                return fmt.Errorf("some tags not found")
            }
        }

        if err := tx.Model(&article).Association("Tags").Replace(&tags); err != nil {
            return err
        }

        return nil
    })
}
```

为什么用 `Replace`？

因为编辑文章时，用户提交的是“最终标签集合”。如果使用 `Append`，旧标签不会被移除。

---

## 七、避免 N+1 的对比

错误写法：

```go
var articles []model.Article
db.Find(&articles)

for i := range articles {
    db.First(&articles[i].User, articles[i].UserID)
    db.First(&articles[i].Category, articles[i].CategoryID)
}
```

如果有 20 篇文章，就会产生：

```text
1 条文章列表 SQL
20 条用户 SQL
20 条分类 SQL
```

正确写法：

```go
db.Preload("User").
    Preload("Category").
    Limit(20).
    Find(&articles)
```

你应该打开 SQL 日志，亲眼看优化前后的 SQL 数量。

---

## 八、常见错误

### 1. 多对多忘记写 tag

错误：

```go
Tags []Tag
```

正确：

```go
Tags []Tag `gorm:"many2many:article_tags;"`
```

### 2. Preload 名字写成表名

错误：

```go
Preload("tags")
```

正确：

```go
Preload("Tags")
```

`Preload` 使用的是结构体字段名，不是数据库表名。

### 3. Replace 没有放进事务

更新文章和替换标签是一个整体业务动作，应该一起成功或一起失败。

---

## 九、本节达标标准

学完本节后，你应该能够做到：

- 写出完整博客关联模型。
- 理解 `article_tags` 中间表的作用。
- 创建文章时绑定标签。
- 查询文章详情时加载作者、分类、标签、评论和评论用户。
- 编辑文章时用 `Replace` 替换标签。
- 解释 `Preload` 字段名和表名的区别。
- 手动制造 N+1 查询，并用 `Preload` 优化。
