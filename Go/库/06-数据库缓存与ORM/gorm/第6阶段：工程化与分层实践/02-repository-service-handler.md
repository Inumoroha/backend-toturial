# 02 Repository、Service、Handler 分层

本节目标：围绕「repository service handler」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

分层的核心目的：让每层只做自己该做的事。

```text
Handler    处理 HTTP 请求和响应
Service    处理业务规则、事务、权限判断
Repository 处理数据库读写
Model      表结构映射
```

## 1. Repository 层

Repository 只关心数据库。

```go
type ArticleRepository struct {
    db *gorm.DB
}

func NewArticleRepository(db *gorm.DB) *ArticleRepository {
    return &ArticleRepository{db: db}
}

func (r *ArticleRepository) Create(ctx context.Context, article *model.Article) error {
    return r.db.WithContext(ctx).Create(article).Error
}

func (r *ArticleRepository) FindByID(ctx context.Context, id uint) (*model.Article, error) {
    var article model.Article
    err := r.db.WithContext(ctx).
        Preload("User").
        Preload("Category").
        Preload("Tags").
        First(&article, id).Error
    if err != nil {
        return nil, err
    }
    return &article, nil
}
```

注意：

- Repository 方法接收 `context.Context`。
- Repository 不处理 HTTP 状态码。
- Repository 不关心请求 JSON。

## 2. Service 层

Service 处理业务逻辑。

```go
type ArticleService struct {
    articleRepo *repository.ArticleRepository
}

func NewArticleService(articleRepo *repository.ArticleRepository) *ArticleService {
    return &ArticleService{articleRepo: articleRepo}
}

func (s *ArticleService) CreateArticle(ctx context.Context, input CreateArticleInput) (*model.Article, error) {
    if input.Title == "" {
        return nil, fmt.Errorf("title required")
    }

    article := &model.Article{
        UserID:     input.UserID,
        CategoryID: input.CategoryID,
        Title:      input.Title,
        Content:    input.Content,
        Status:     "draft",
    }

    if err := s.articleRepo.Create(ctx, article); err != nil {
        return nil, err
    }

    return article, nil
}
```

Service 适合放：

- 参数业务校验
- 权限判断
- 状态流转
- 多 Repository 协作
- 事务

## 3. Handler 层

以 Gin 为例：

```go
type ArticleHandler struct {
    articleService *service.ArticleService
}

func NewArticleHandler(articleService *service.ArticleService) *ArticleHandler {
    return &ArticleHandler{articleService: articleService}
}

func (h *ArticleHandler) Create(c *gin.Context) {
    var req CreateArticleRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    article, err := h.articleService.CreateArticle(c.Request.Context(), service.CreateArticleInput{
        UserID:     req.UserID,
        CategoryID: req.CategoryID,
        Title:      req.Title,
        Content:    req.Content,
    })
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    c.JSON(http.StatusCreated, article)
}
```

Handler 适合放：

- 解析请求
- 调用 Service
- 返回响应

不适合放：

- 复杂 SQL
- 事务
- 大量业务判断

## 4. 事务放在哪里

事务通常放在 Service 层，因为事务往往代表一个业务用例。

例如编辑文章：

```text
1. 更新文章
2. 替换标签
```

这是一个业务动作，应该由 Service 协调。

## 5. Repository 如何支持事务

一种常见做法是让 Repository 支持传入 `*gorm.DB`：

```go
func (r *ArticleRepository) WithDB(db *gorm.DB) *ArticleRepository {
    return &ArticleRepository{db: db}
}
```

Service 里：

```go
err := db.Transaction(func(tx *gorm.DB) error {
    articleRepo := s.articleRepo.WithDB(tx)
    return articleRepo.Update(ctx, article)
})
```

也可以抽象得更精致，但初学阶段先保持清晰。

## 本节练习

- [ ] 为 `Article` 写 Repository。
- [ ] 为 `Article` 写 Service。
- [ ] 为 `Article` 写 Handler。
- [ ] 实现创建文章接口。
- [ ] 实现文章详情接口。
- [ ] 确认 Handler 中没有直接调用 GORM。
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
