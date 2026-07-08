# 04 高级特性综合练习

本节目标：把 Hook、Scope、Context、自定义 JSON 字段和锁机制放进真实业务中练习。高级特性不要孤立学习，要放到业务问题里理解。

---

## 一、练习场景

我们给博客系统增加几个需求：

```text
1. 用户创建时自动生成 UUID
2. 文章列表复用分页 Scope
3. 文章增加 JSON 扩展信息
4. 查询必须带 Context
5. 阅读量更新使用表达式
6. 库存类场景演示乐观锁
```

---

## 二、Hook：自动生成 UUID

模型：

```go
type User struct {
    ID        uint
    UUID      string `gorm:"size:36;not null;uniqueIndex"`
    Username  string
    Email     string
    CreatedAt time.Time
}
```

Hook：

```go
func (u *User) BeforeCreate(tx *gorm.DB) error {
    if u.UUID != "" {
        return nil
    }

    u.UUID = uuid.NewString()
    return nil
}
```

需要依赖：

```powershell
go get github.com/google/uuid
```

适合放在 Hook 里的逻辑应该轻量、稳定、不依赖复杂外部系统。

---

## 三、Scope：分页复用

```go
func Paginate(page, pageSize int) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if page < 1 {
            page = 1
        }
        if pageSize < 1 {
            pageSize = 10
        }
        if pageSize > 100 {
            pageSize = 100
        }
        return db.Offset((page - 1) * pageSize).Limit(pageSize)
    }
}
```

使用：

```go
err := db.Scopes(Paginate(page, pageSize)).
    Where("status = ?", "published").
    Order("created_at DESC").
    Find(&articles).Error
```

Scope 适合封装可复用查询条件，不要把复杂业务流程塞进去。

---

## 四、Scope：业务状态过滤

```go
func Published(db *gorm.DB) *gorm.DB {
    return db.Where("status = ?", "published")
}
```

组合使用：

```go
db.Scopes(Published, Paginate(page, pageSize)).Find(&articles)
```

这种写法能减少重复的 `Where("status = ?", "published")`。

---

## 五、自定义 JSON 字段

文章扩展信息：

```go
type ArticleMeta struct {
    CoverURL string   `json:"cover_url"`
    Keywords []string `json:"keywords"`
}

type Article struct {
    ID    uint
    Title string
    Meta  ArticleMeta `gorm:"serializer:json"`
}
```

写入：

```go
article := Article{
    Title: "GORM JSON 字段",
    Meta: ArticleMeta{
        CoverURL: "https://example.com/cover.png",
        Keywords: []string{"GORM", "Go", "ORM"},
    },
}

db.Create(&article)
```

读取时，GORM 会把 JSON 反序列化回 `ArticleMeta`。

---

## 六、Context：Repository 标准写法

```go
func (r *ArticleRepository) FindByID(ctx context.Context, id uint) (*model.Article, error) {
    var article model.Article
    err := r.db.WithContext(ctx).First(&article, id).Error
    if err != nil {
        return nil, err
    }
    return &article, nil
}
```

Handler 传入：

```go
article, err := h.articleService.FindByID(c.Request.Context(), id)
```

好处：

- 请求取消时，数据库操作也能感知。
- 可以设置超时。
- 方便链路追踪和日志。

---

## 七、阅读量更新

不要先查再加：

```go
article.ViewCount++
db.Save(&article)
```

更推荐表达式更新：

```go
result := db.Model(&model.Article{}).
    Where("id = ?", articleID).
    Update("view_count", gorm.Expr("view_count + ?", 1))
```

这条 SQL 在数据库内完成自增，更适合并发场景。

---

## 八、乐观锁示例

模型：

```go
type Product struct {
    ID      uint
    Name    string
    Stock   int
    Version int
}
```

更新：

```go
func ReduceStockWithVersion(db *gorm.DB, productID uint, quantity int) error {
    var product Product
    if err := db.First(&product, productID).Error; err != nil {
        return err
    }

    result := db.Model(&Product{}).
        Where("id = ? AND version = ? AND stock >= ?", product.ID, product.Version, quantity).
        Updates(map[string]any{
            "stock":   gorm.Expr("stock - ?", quantity),
            "version": gorm.Expr("version + 1"),
        })

    if result.Error != nil {
        return result.Error
    }
    if result.RowsAffected == 0 {
        return fmt.Errorf("stock conflict or not enough")
    }

    return nil
}
```

乐观锁适合冲突不太高的场景。冲突高时，可能需要重试或换方案。

---

## 九、常见错误

### 1. Hook 里做复杂业务

Hook 不适合发消息、调接口、写复杂事务。复杂业务放 Service。

### 2. Scope 过度封装

Scope 应该短小清楚。太复杂的查询直接写 Repository 方法更好。

### 3. Context 只在 Handler 使用

Context 应该一路传到 Repository，并进入 GORM：

```go
db.WithContext(ctx)
```

### 4. JSON 字段滥用

JSON 字段适合扩展信息，不适合替代关系建模。需要频繁筛选和关联的数据，还是应该建表。

---

## 十、本节达标标准

学完本节后，你应该能够做到：

- 使用 Hook 自动填充 UUID。
- 使用 Scope 复用分页和状态过滤。
- 使用 `serializer:json` 保存扩展信息。
- 在 Repository 中标准传递 Context。
- 使用表达式更新阅读量。
- 写出一个乐观锁更新示例。
