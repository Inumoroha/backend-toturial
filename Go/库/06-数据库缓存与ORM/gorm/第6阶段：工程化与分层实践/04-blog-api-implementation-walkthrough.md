# 04 博客 API 分层实现 walkthrough

本节目标：按照 PostgreSQL 项目教程那样，把一个接口从请求、DTO、Handler、Service、Repository 到 GORM 查询完整走一遍。这里选择“创建文章接口”，因为它同时涉及参数校验、事务、模型、标签关联和错误处理。

---

## 一、接口设计

接口：

```http
POST /api/articles
Content-Type: application/json
Authorization: Bearer <token>
```

请求体：

```json
{
  "category_id": 1,
  "title": "GORM 入门",
  "summary": "这是一篇 GORM 入门文章",
  "content": "正文内容",
  "tag_ids": [1, 2, 3]
}
```

响应：

```json
{
  "id": 100,
  "title": "GORM 入门",
  "summary": "这是一篇 GORM 入门文章",
  "status": "draft",
  "created_at": "2026-07-05T10:00:00+08:00"
}
```

---

## 二、文件结构

```text
internal
├── handler
│   └── article_handler.go
├── service
│   └── article_service.go
├── repository
│   ├── article_repository.go
│   ├── category_repository.go
│   └── tag_repository.go
└── dto
    └── article.go
```

---

## 三、DTO 定义

`internal/dto/article.go`：

```go
package dto

import "time"

type CreateArticleRequest struct {
    CategoryID uint   `json:"category_id" binding:"required"`
    Title      string `json:"title" binding:"required"`
    Summary    string `json:"summary"`
    Content    string `json:"content" binding:"required"`
    TagIDs     []uint `json:"tag_ids"`
}

type ArticleResponse struct {
    ID        uint      `json:"id"`
    Title     string    `json:"title"`
    Summary   string    `json:"summary"`
    Status    string    `json:"status"`
    CreatedAt time.Time `json:"created_at"`
}
```

为什么不用 `model.Article` 直接绑定请求？

因为请求体不应该决定数据库模型的所有字段。例如 `UserID` 应该来自登录态，而不是前端随便传。

---

## 四、Repository：创建文章和替换标签

`internal/repository/article_repository.go`：

```go
type ArticleRepository struct {
    db *gorm.DB
}

func NewArticleRepository(db *gorm.DB) *ArticleRepository {
    return &ArticleRepository{db: db}
}

func (r *ArticleRepository) WithDB(db *gorm.DB) *ArticleRepository {
    return &ArticleRepository{db: db}
}

func (r *ArticleRepository) Create(ctx context.Context, article *model.Article) error {
    return r.db.WithContext(ctx).Create(article).Error
}

func (r *ArticleRepository) ReplaceTags(ctx context.Context, article *model.Article, tags []model.Tag) error {
    return r.db.WithContext(ctx).Model(article).Association("Tags").Replace(&tags)
}
```

`WithDB` 用于事务。Service 开启事务后，把 `tx` 传给 Repository。

---

## 五、Repository：查询分类和标签

```go
type CategoryRepository struct {
    db *gorm.DB
}

func (r *CategoryRepository) Exists(ctx context.Context, id uint) (bool, error) {
    var count int64
    err := r.db.WithContext(ctx).
        Model(&model.Category{}).
        Where("id = ?", id).
        Count(&count).Error
    return count > 0, err
}
```

标签：

```go
type TagRepository struct {
    db *gorm.DB
}

func (r *TagRepository) FindByIDs(ctx context.Context, ids []uint) ([]model.Tag, error) {
    var tags []model.Tag
    if len(ids) == 0 {
        return tags, nil
    }

    err := r.db.WithContext(ctx).
        Where("id IN ?", ids).
        Find(&tags).Error
    return tags, err
}
```

---

## 六、Service：组织业务和事务

```go
type CreateArticleInput struct {
    UserID     uint
    CategoryID uint
    Title      string
    Summary    string
    Content    string
    TagIDs     []uint
}

type ArticleService struct {
    db           *gorm.DB
    articleRepo  *repository.ArticleRepository
    categoryRepo *repository.CategoryRepository
    tagRepo      *repository.TagRepository
}

func (s *ArticleService) CreateArticle(ctx context.Context, input CreateArticleInput) (*model.Article, error) {
    if strings.TrimSpace(input.Title) == "" {
        return nil, ErrInvalidInput
    }
    if strings.TrimSpace(input.Content) == "" {
        return nil, ErrInvalidInput
    }

    exists, err := s.categoryRepo.Exists(ctx, input.CategoryID)
    if err != nil {
        return nil, err
    }
    if !exists {
        return nil, ErrCategoryNotFound
    }

    tags, err := s.tagRepo.FindByIDs(ctx, input.TagIDs)
    if err != nil {
        return nil, err
    }
    if len(tags) != len(input.TagIDs) {
        return nil, ErrTagNotFound
    }

    article := &model.Article{
        UserID:     input.UserID,
        CategoryID: input.CategoryID,
        Title:      strings.TrimSpace(input.Title),
        Summary:    strings.TrimSpace(input.Summary),
        Content:    input.Content,
        Status:     "draft",
    }

    err = s.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        articleRepo := s.articleRepo.WithDB(tx)

        if err := articleRepo.Create(ctx, article); err != nil {
            return err
        }

        if err := articleRepo.ReplaceTags(ctx, article, tags); err != nil {
            return err
        }

        return nil
    })
    if err != nil {
        return nil, err
    }

    return article, nil
}
```

这里事务保护的是：

```text
创建文章
写入 article_tags 关联
```

如果标签关联失败，文章也应该回滚。

---

## 七、Handler：解析请求和返回响应

```go
type ArticleHandler struct {
    articleService *service.ArticleService
}

func (h *ArticleHandler) Create(c *gin.Context) {
    var req dto.CreateArticleRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid request"})
        return
    }

    userID := getCurrentUserID(c)

    article, err := h.articleService.CreateArticle(c.Request.Context(), service.CreateArticleInput{
        UserID:     userID,
        CategoryID: req.CategoryID,
        Title:      req.Title,
        Summary:    req.Summary,
        Content:    req.Content,
        TagIDs:     req.TagIDs,
    })
    if err != nil {
        writeAppError(c, err)
        return
    }

    c.JSON(http.StatusCreated, dto.ArticleResponse{
        ID:        article.ID,
        Title:     article.Title,
        Summary:   article.Summary,
        Status:    article.Status,
        CreatedAt: article.CreatedAt,
    })
}
```

`getCurrentUserID` 可以先写成临时函数：

```go
func getCurrentUserID(c *gin.Context) uint {
    return 1
}
```

后续接入 JWT 后，再从 token 中解析用户 ID。

---

## 八、手动测试

先准备分类和标签：

```sql
INSERT INTO categories (name, created_at, updated_at)
VALUES ('Go 后端', NOW(), NOW());

INSERT INTO tags (name, created_at, updated_at)
VALUES ('GORM', NOW(), NOW()), ('Go', NOW(), NOW());
```

请求：

```powershell
Invoke-RestMethod `
  -Method Post `
  -Uri "http://localhost:8080/api/articles" `
  -ContentType "application/json" `
  -Body '{"category_id":1,"title":"GORM 入门","summary":"GORM 基础","content":"正文","tag_ids":[1,2]}'
```

检查中间表：

```sql
SELECT *
FROM article_tags;
```

应该能看到文章 ID 和标签 ID 的绑定关系。

---

## 九、常见错误

### 1. 前端传 UserID

不要让前端决定作者是谁。作者应该来自登录态。

### 2. Handler 直接操作 GORM

Handler 里不要写：

```go
db.Create(&article)
```

它应该调用 Service。

### 3. 创建文章和绑定标签没有事务

这会导致文章创建成功但标签绑定失败，留下不完整数据。

### 4. 没校验分类和标签是否存在

不要盲目相信前端传来的 ID。

---

## 十、本节达标标准

学完本节后，你应该能够做到：

- 设计创建文章接口的请求和响应。
- 区分 Request DTO、Service Input 和 GORM Model。
- 在 Repository 中封装 GORM 操作。
- 在 Service 中组织事务。
- 在 Handler 中只处理 HTTP 输入输出。
- 创建文章时正确写入多对多标签关系。
