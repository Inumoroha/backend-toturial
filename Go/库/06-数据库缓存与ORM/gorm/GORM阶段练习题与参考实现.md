# GORM 阶段练习题与参考实现

本节目标：给 00 到 09 每个阶段配套练习题和参考实现，让你不只是阅读教程，而是能写出代码、观察 SQL、解释结果、复盘问题。

练习原则：

```text
先自己写
再运行观察 SQL
再对照参考实现
最后写复盘
```

建议项目：

- 博客系统：适合练习模型、CRUD、查询、关联、工程分层。
- 订单系统：适合练习事务、库存、一致性、并发和状态机。

---

## 第 0 阶段：准备阶段与本地实验室

### 练习 0-1：搭建本地 GORM 实验项目

目标：

- 初始化 Go 项目。
- 安装 GORM 和数据库驱动。
- 能连接本地数据库。

任务：

```text
1. 新建 go module
2. 安装 gorm.io/gorm
3. 安装 PostgreSQL 或 MySQL 驱动
4. 编写 database.InitDB
5. 启动程序时 Ping 数据库
```

参考实现：

```go
package database

import (
    "database/sql"
    "time"

    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

func Open(dsn string) (*gorm.DB, error) {
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        return nil, err
    }

    sqlDB, err := db.DB()
    if err != nil {
        return nil, err
    }

    configurePool(sqlDB)

    if err := sqlDB.Ping(); err != nil {
        return nil, err
    }

    return db, nil
}

func configurePool(db *sql.DB) {
    db.SetMaxOpenConns(20)
    db.SetMaxIdleConns(5)
    db.SetConnMaxLifetime(time.Hour)
}
```

达标标准：

- 程序启动时能成功连接数据库。
- 连接失败时返回明确错误。
- 能说出连接池三个参数的含义。

常见坑：

- 每个请求都调用 `gorm.Open`。
- DSN 写死在代码里。
- 忘记检查 `Ping` 错误。

---

### 练习 0-2：打开 SQL 日志

目标：

- 能看到 GORM 生成的 SQL。
- 为后续所有阶段养成观察 SQL 的习惯。

参考实现：

```go
db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
    Logger: logger.Default.LogMode(logger.Info),
})
```

达标标准：

- 执行查询时能在控制台看到 SQL。
- 能区分 SQL 文本和参数。
- 能判断一次接口执行了几条 SQL。

---

## 第 1 阶段：GORM 入门与单表 CRUD

### 练习 1-1：实现用户注册

目标：

- 使用 `Create` 插入用户。
- 理解主键回填。
- 处理唯一冲突。

模型：

```go
type User struct {
    ID           uint
    Username     string `gorm:"size:64;not null;uniqueIndex"`
    Email        string `gorm:"size:128;not null;uniqueIndex"`
    PasswordHash string `gorm:"size:255;not null"`
    CreatedAt    time.Time
    UpdatedAt    time.Time
}
```

参考实现：

```go
func (r *UserRepository) Create(ctx context.Context, user *User) error {
    return r.db.WithContext(ctx).Create(user).Error
}
```

测试点：

```text
1. 插入成功后 user.ID 有值
2. 重复邮箱插入失败
3. SQL 中出现 INSERT
```

达标标准：

- 会用指针接收主键回填。
- 知道唯一冲突要转成业务错误。

---

### 练习 1-2：实现用户查询、更新和删除

目标：

- 掌握 `First`、`Updates`、`Delete`。
- 正确处理不存在。
- 理解零值更新。

参考实现：

```go
func (r *UserRepository) FindByID(ctx context.Context, id uint) (*User, error) {
    var user User
    err := r.db.WithContext(ctx).First(&user, id).Error
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return nil, nil
    }
    if err != nil {
        return nil, err
    }
    return &user, nil
}

func (r *UserRepository) UpdateProfile(ctx context.Context, id uint, nickname string, age int) error {
    return r.db.WithContext(ctx).
        Model(&User{}).
        Where("id = ?", id).
        Updates(map[string]any{
            "nickname": nickname,
            "age":      age,
        }).Error
}

func (r *UserRepository) Delete(ctx context.Context, id uint) error {
    return r.db.WithContext(ctx).Delete(&User{}, id).Error
}
```

达标标准：

- 查询不存在不当成系统错误。
- 更新空字符串和 0 能生效。
- 能说明软删除和物理删除的区别。

---

## 第 2 阶段：模型设计与字段映射

### 练习 2-1：设计博客核心模型

目标：

- 从业务关系推导模型。
- 使用索引、唯一约束和字段类型。

任务：

```text
1. 设计 User
2. 设计 Category
3. 设计 Article
4. 设计 Comment
5. 给高频查询字段加索引
```

参考实现：

```go
type Article struct {
    ID         uint
    UserID     uint      `gorm:"not null;index"`
    CategoryID uint      `gorm:"not null;index:idx_article_category_status_created"`
    Title      string    `gorm:"size:200;not null"`
    Summary    string    `gorm:"size:500"`
    Content    string    `gorm:"type:text;not null"`
    Status     string    `gorm:"size:32;not null;index:idx_article_category_status_created"`
    CreatedAt  time.Time `gorm:"index:idx_article_category_status_created"`
    UpdatedAt  time.Time
    DeletedAt  gorm.DeletedAt `gorm:"index"`
}
```

达标标准：

- 能解释每个索引为什么存在。
- 能说明 `Content` 为什么不适合在列表页查询。
- 能说明 `DeletedAt` 对查询和唯一索引的影响。

---

### 练习 2-2：Model 和 DTO 分离

目标：

- 避免敏感字段泄露。
- 理解数据库模型和接口结构的差异。

参考实现：

```go
type UserResponse struct {
    ID       uint   `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email"`
}

func ToUserResponse(user User) UserResponse {
    return UserResponse{
        ID:       user.ID,
        Username: user.Username,
        Email:    user.Email,
    }
}
```

达标标准：

- 响应中不出现 `PasswordHash`。
- DTO 不依赖 GORM tag。
- 能解释为什么不能直接返回 Model。

---

## 第 3 阶段：查询能力进阶

### 练习 3-1：实现文章列表动态查询

目标：

- 支持分类、状态、关键词、时间范围筛选。
- 支持分页和排序。
- 避免 SQL 注入。

参考实现：

```go
type ArticleQuery struct {
    CategoryID uint
    Status     string
    Keyword    string
    StartTime  *time.Time
    EndTime    *time.Time
    Page       int
    PageSize   int
}

func (r *ArticleRepository) List(ctx context.Context, q ArticleQuery) ([]Article, int64, error) {
    if q.Page <= 0 {
        q.Page = 1
    }
    if q.PageSize <= 0 || q.PageSize > 100 {
        q.PageSize = 20
    }

    db := r.db.WithContext(ctx).Model(&Article{})

    if q.CategoryID > 0 {
        db = db.Where("category_id = ?", q.CategoryID)
    }
    if q.Status != "" {
        db = db.Where("status = ?", q.Status)
    }
    if q.Keyword != "" {
        db = db.Where("title LIKE ?", "%"+q.Keyword+"%")
    }
    if q.StartTime != nil {
        db = db.Where("created_at >= ?", *q.StartTime)
    }
    if q.EndTime != nil {
        db = db.Where("created_at < ?", *q.EndTime)
    }

    var total int64
    if err := db.Count(&total).Error; err != nil {
        return nil, 0, err
    }

    var articles []Article
    err := db.Select("id", "category_id", "title", "summary", "status", "created_at").
        Order("created_at DESC").
        Limit(q.PageSize).
        Offset((q.Page - 1) * q.PageSize).
        Find(&articles).Error

    return articles, total, err
}
```

达标标准：

- 动态条件不会覆盖前面的条件。
- 列表页不查询 `content`。
- 页大小有上限。
- 排序不直接使用用户输入。

---

### 练习 3-2：实现统计查询

目标：

- 使用 `Group` 和 `Scan`。
- 把统计结果映射到 DTO。

参考实现：

```go
type ArticleStatusCount struct {
    Status string
    Count  int64
}

func (r *ArticleRepository) CountByStatus(ctx context.Context) ([]ArticleStatusCount, error) {
    var result []ArticleStatusCount
    err := r.db.WithContext(ctx).
        Model(&Article{}).
        Select("status, count(*) as count").
        Group("status").
        Scan(&result).Error
    return result, err
}
```

达标标准：

- 能解释为什么这里用 `Scan`。
- 能说出生成 SQL 大概是什么样。

---

## 第 4 阶段：关联关系与预加载

### 练习 4-1：实现文章详情预加载

目标：

- 使用 `Preload` 加载作者、分类、标签。
- 避免 N+1 查询。

参考实现：

```go
func (r *ArticleRepository) Detail(ctx context.Context, id uint) (*Article, error) {
    var article Article
    err := r.db.WithContext(ctx).
        Preload("User").
        Preload("Category").
        Preload("Tags").
        First(&article, id).Error

    if errors.Is(err, gorm.ErrRecordNotFound) {
        return nil, nil
    }
    if err != nil {
        return nil, err
    }
    return &article, nil
}
```

达标标准：

- `Preload` 使用结构体字段名。
- 能通过 SQL 日志确认不是循环查询。
- 能说明文章详情和文章列表加载关联的差异。

---

### 练习 4-2：替换文章标签

目标：

- 使用 Association Mode。
- 理解多对多中间表。
- 使用事务保证一致性。

参考实现：

```go
func (s *ArticleService) ReplaceTags(ctx context.Context, articleID uint, tagIDs []uint) error {
    return s.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        var article Article
        if err := tx.First(&article, articleID).Error; err != nil {
            return err
        }

        var tags []Tag
        if len(tagIDs) > 0 {
            if err := tx.Where("id IN ?", tagIDs).Find(&tags).Error; err != nil {
                return err
            }
            if len(tags) != len(tagIDs) {
                return ErrTagNotFound
            }
        }

        return tx.Model(&article).Association("Tags").Replace(tags)
    })
}
```

达标标准：

- 标签不存在时不能静默忽略。
- 替换标签和文章校验在同一事务中。
- 能说明 `Replace` 删除的是关联关系，不是标签本身。

---

## 第 5 阶段：事务并发与数据一致性

### 练习 5-1：实现创建订单

目标：

- 设计订单主表和明细表。
- 在事务中写入多张表。
- 安全扣减库存。

参考实现：

```go
func (s *OrderService) CreateOrder(ctx context.Context, req CreateOrderRequest) error {
    return s.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        order := Order{
            UserID: req.UserID,
            Status: "pending",
        }
        if err := tx.Create(&order).Error; err != nil {
            return err
        }

        for _, item := range req.Items {
            result := tx.Model(&Product{}).
                Where("id = ? AND stock >= ?", item.ProductID, item.Quantity).
                UpdateColumn("stock", gorm.Expr("stock - ?", item.Quantity))
            if result.Error != nil {
                return result.Error
            }
            if result.RowsAffected == 0 {
                return ErrInsufficientStock
            }

            orderItem := OrderItem{
                OrderID:   order.ID,
                ProductID: item.ProductID,
                Quantity:  item.Quantity,
            }
            if err := tx.Create(&orderItem).Error; err != nil {
                return err
            }
        }

        return nil
    })
}
```

达标标准：

- 订单、明细、库存都在同一事务。
- 库存扣减使用条件更新。
- 检查 `RowsAffected`。
- 任一步失败都回滚。

---

### 练习 5-2：实现订单取消恢复库存

目标：

- 状态机更新。
- 防止重复取消。
- 恢复库存。

参考实现：

```go
func (s *OrderService) CancelOrder(ctx context.Context, orderID uint) error {
    return s.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        result := tx.Model(&Order{}).
            Where("id = ? AND status = ?", orderID, "pending").
            Update("status", "cancelled")
        if result.Error != nil {
            return result.Error
        }
        if result.RowsAffected == 0 {
            return ErrOrderCannotCancel
        }

        var items []OrderItem
        if err := tx.Where("order_id = ?", orderID).Find(&items).Error; err != nil {
            return err
        }

        for _, item := range items {
            if err := tx.Model(&Product{}).
                Where("id = ?", item.ProductID).
                UpdateColumn("stock", gorm.Expr("stock + ?", item.Quantity)).Error; err != nil {
                return err
            }
        }

        return nil
    })
}
```

达标标准：

- 只有待支付订单能取消。
- 重复取消不会重复恢复库存。
- 所有操作使用 `tx`。

---

## 第 6 阶段：工程化与分层实践

### 练习 6-1：拆分 Handler、Service、Repository

目标：

- 明确职责边界。
- 让事务边界在 Service 层。

目录建议：

```text
internal/
  handler/
  service/
  repository/
  model/
  dto/
  database/
```

参考接口：

```go
type ArticleRepository interface {
    Create(ctx context.Context, tx *gorm.DB, article *Article) error
    FindByID(ctx context.Context, id uint) (*Article, error)
    List(ctx context.Context, q ArticleQuery) ([]Article, int64, error)
}

type ArticleService struct {
    db   *gorm.DB
    repo ArticleRepository
}
```

达标标准：

- Handler 不直接调用 GORM。
- Repository 不处理 HTTP。
- Service 可以开启事务并传递 `tx`。

---

### 练习 6-2：统一错误转换

目标：

- 区分数据库错误和业务错误。
- 避免 Handler 中到处判断 GORM 错误。

参考实现：

```go
var ErrArticleNotFound = errors.New("article not found")

func (r *ArticleRepository) FindByID(ctx context.Context, id uint) (*Article, error) {
    var article Article
    err := r.db.WithContext(ctx).First(&article, id).Error
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return nil, ErrArticleNotFound
    }
    if err != nil {
        return nil, err
    }
    return &article, nil
}
```

达标标准：

- 业务层不用直接依赖所有 GORM 错误。
- Handler 能把业务错误映射成 HTTP 状态码。

---

## 第 7 阶段：性能分析与问题排查

### 练习 7-1：制造并修复 N+1

目标：

- 亲眼看到 N+1。
- 使用 `Preload` 修复。

错误写法：

```go
var articles []Article
db.Find(&articles)

for i := range articles {
    db.Where("article_id = ?", articles[i].ID).Find(&articles[i].Comments)
}
```

修复写法：

```go
var articles []Article
db.Preload("Comments").Find(&articles)
```

达标标准：

- 能通过 SQL 日志数出修复前后的 SQL 条数。
- 能说明为什么循环查关联会慢。

---

### 练习 7-2：优化文章列表慢查询

目标：

- 使用字段裁剪。
- 使用索引。
- 使用 `EXPLAIN`。

优化前：

```go
db.Where("status = ?", "published").
    Order("created_at DESC").
    Find(&articles)
```

优化后：

```go
db.Select("id", "title", "summary", "status", "created_at").
    Where("status = ?", "published").
    Order("created_at DESC").
    Limit(20).
    Find(&articles)
```

索引建议：

```go
Status    string    `gorm:"index:idx_status_created"`
CreatedAt time.Time `gorm:"index:idx_status_created"`
```

达标标准：

- 列表不查询正文。
- 有分页限制。
- 能执行 `EXPLAIN` 并说明是否命中索引。

---

## 第 8 阶段：GORM 高级特性

### 练习 8-1：使用 Scope 封装分页

目标：

- 复用分页逻辑。
- 让查询链更清晰。

参考实现：

```go
func Paginate(page, pageSize int) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        if page <= 0 {
            page = 1
        }
        if pageSize <= 0 || pageSize > 100 {
            pageSize = 20
        }
        return db.Offset((page - 1) * pageSize).Limit(pageSize)
    }
}

db.Scopes(Paginate(req.Page, req.PageSize)).Find(&articles)
```

达标标准：

- Scope 不执行查询，只返回修改后的 `db`。
- 分页参数有默认值和上限。

---

### 练习 8-2：使用 Context 控制查询超时

目标：

- 理解请求取消和数据库查询取消。
- 在 Repository 中使用 `WithContext`。

参考实现：

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

err := db.WithContext(ctx).Where("status = ?", "published").Find(&articles).Error
```

达标标准：

- Repository 方法接收 `context.Context`。
- 所有 GORM 操作使用 `WithContext`。
- 能解释请求取消后为什么数据库查询也应该停止。

---

### 练习 8-3：使用 Hook 设置默认值

目标：

- 理解 Hook 适合做轻量模型逻辑。

参考实现：

```go
func (a *Article) BeforeCreate(tx *gorm.DB) error {
    if a.Status == "" {
        a.Status = "draft"
    }
    return nil
}
```

达标标准：

- Hook 中不调用外部服务。
- Hook 不承载复杂业务流程。
- 能说明 Hook 的隐式副作用风险。

---

## 第 9 阶段：项目落地：博客与订单系统

### 练习 9-1：完成博客系统最小闭环

目标：

- 把前面模型、查询、关联、事务、分层整合起来。

功能清单：

```text
1. 用户注册
2. 创建分类
3. 创建标签
4. 发布文章
5. 文章绑定标签
6. 文章列表分页
7. 文章详情预加载
8. 评论文章
9. 软删除文章
```

达标标准：

- 所有接口走 Handler、Service、Repository。
- 文章发布和标签绑定在事务中。
- 列表接口不查询正文。
- 详情接口使用 `Preload`。
- SQL 日志中没有明显 N+1。

---

### 练习 9-2：完成订单系统最小闭环

目标：

- 练习事务、一致性、状态机和幂等。

功能清单：

```text
1. 创建商品
2. 查询商品库存
3. 创建订单
4. 扣减库存
5. 支付订单
6. 重复支付幂等
7. 取消订单
8. 恢复库存
```

达标标准：

- 创建订单使用事务。
- 扣库存使用条件更新。
- 支付使用状态机和唯一流水号。
- 取消订单不会重复恢复库存。
- 至少编写库存不足和重复支付两个测试。

---

## 十、最终综合练习

### 综合题：实现“发布文章并扣减积分”

业务背景：

用户发布文章需要扣减 10 积分，同时文章可以绑定多个标签。要求文章创建、积分扣减、标签绑定必须一致成功或一致失败。

任务：

```text
1. 用户积分不足时不能发布文章
2. 创建文章
3. 扣减用户积分
4. 绑定标签
5. 任一步失败全部回滚
```

参考实现：

```go
func (s *ArticleService) PublishWithPoints(ctx context.Context, req PublishRequest) error {
    return s.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        result := tx.Model(&User{}).
            Where("id = ? AND points >= ?", req.UserID, 10).
            UpdateColumn("points", gorm.Expr("points - ?", 10))
        if result.Error != nil {
            return result.Error
        }
        if result.RowsAffected == 0 {
            return ErrInsufficientPoints
        }

        article := Article{
            UserID:  req.UserID,
            Title:   req.Title,
            Content: req.Content,
            Status:  "published",
        }
        if err := tx.Create(&article).Error; err != nil {
            return err
        }

        var tags []Tag
        if len(req.TagIDs) > 0 {
            if err := tx.Where("id IN ?", req.TagIDs).Find(&tags).Error; err != nil {
                return err
            }
            if len(tags) != len(req.TagIDs) {
                return ErrTagNotFound
            }
        }

        if err := tx.Model(&article).Association("Tags").Replace(tags); err != nil {
            return err
        }

        return nil
    })
}
```

达标标准：

- 积分扣减是条件更新。
- 文章创建和标签绑定在同一事务。
- 标签不存在时回滚积分扣减和文章创建。
- 所有事务内操作使用 `tx`。

---

## 十一、练习复盘模板

每做完一个练习，写一段复盘：

```text
练习名称：
我写了哪些模型：
我用了哪些 GORM API：
生成了哪些关键 SQL：
我遇到的问题：
我是怎么定位的：
最后怎么修复：
这个练习对应哪些面试题：
```

示例：

```text
练习名称：创建订单并扣减库存
我用了哪些 GORM API：Transaction、Create、Model、Where、UpdateColumn、RowsAffected
生成了哪些关键 SQL：INSERT orders、UPDATE products SET stock = stock - ? WHERE id = ? AND stock >= ?
我遇到的问题：一开始先查询库存再扣减，意识到并发下可能超卖
最后怎么修复：改成条件更新，并检查 RowsAffected
对应面试题：事务怎么用、如何安全扣库存、RowsAffected 有什么用
```
