# GORM 模拟面试卷

本节目标：通过模拟面试卷训练你把 GORM 知识讲出来，而不是只停留在“看过文档”。每套卷子都包含题目、答题要求、评分标准和参考回答要点。

建议使用方式：

```text
第一遍：闭卷口述
第二遍：写出关键代码
第三遍：对照评分标准补充遗漏
第四遍：把答案改成自己的项目经历
```

---

## 一、初级模拟面试卷

适用对象：刚学完 GORM 入门、模型设计、单表 CRUD、基础查询。

建议时长：45 分钟。

总分：100 分。

### 题目 1：请介绍一下 GORM 是什么，以及它解决了什么问题。

分值：10 分。

答题要求：

- 解释 ORM。
- 说明结构体和数据库表的映射。
- 说明 GORM 不能替代 SQL。

参考要点：

GORM 是 Go 语言常用 ORM，能把 Go 结构体映射到数据库表，把字段映射到列。它简化 CRUD、关联、事务等操作，提高开发效率。但开发者仍要理解 SQL、索引和事务，否则容易写出低效查询。

扣分点：

- 只说“GORM 是操作数据库的库”。
- 说“用了 GORM 就不用学 SQL”。

---

### 题目 2：请写出一个用户模型，包含 ID、用户名、邮箱、密码哈希、创建时间、更新时间和软删除字段。

分值：12 分。

参考实现：

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

评分标准：

- 主键正确：2 分。
- 唯一索引正确：3 分。
- 密码字段不直接叫 `Password`：2 分。
- 时间字段正确：2 分。
- 软删除字段正确：3 分。

追问：

- 为什么响应 DTO 不能直接返回这个模型？

---

### 题目 3：`First`、`Take`、`Last`、`Find` 有什么区别？

分值：10 分。

参考要点：

- `First`：按主键升序取第一条。
- `Last`：按主键降序取最后一条。
- `Take`：不指定排序取一条。
- `Find`：查询多条。
- 单条查询没有结果时要处理 `gorm.ErrRecordNotFound`。

扣分点：

- 说不出排序差异。
- 不知道 `Find` 查询空切片通常不是错误。

---

### 题目 4：请实现一个根据邮箱查询用户的 Repository 方法。

分值：12 分。

参考实现：

```go
func (r *UserRepository) FindByEmail(ctx context.Context, email string) (*User, error) {
    var user User
    err := r.db.WithContext(ctx).
        Where("email = ?", email).
        First(&user).Error

    if errors.Is(err, gorm.ErrRecordNotFound) {
        return nil, nil
    }
    if err != nil {
        return nil, err
    }
    return &user, nil
}
```

评分标准：

- 使用参数绑定：3 分。
- 使用 `WithContext`：2 分。
- 正确处理不存在：3 分。
- 返回指针和错误语义清晰：2 分。
- Repository 不处理 HTTP 状态码：2 分。

---

### 题目 5：为什么使用结构体 `Updates` 时零值字段可能不更新？如何解决？

分值：10 分。

参考要点：

结构体更新时，GORM 默认忽略零值，例如空字符串、0、false。解决方式是使用 `map`，或者使用 `Select` 明确指定字段。

参考实现：

```go
db.Model(&user).Updates(map[string]any{
    "nickname": "",
    "age":      0,
})
```

---

### 题目 6：请实现文章列表分页查询，要求支持状态筛选和按创建时间倒序。

分值：16 分。

参考实现：

```go
type ArticleListQuery struct {
    Status   string
    Page     int
    PageSize int
}

func (r *ArticleRepository) List(ctx context.Context, q ArticleListQuery) ([]Article, int64, error) {
    if q.Page <= 0 {
        q.Page = 1
    }
    if q.PageSize <= 0 || q.PageSize > 100 {
        q.PageSize = 20
    }

    query := r.db.WithContext(ctx).Model(&Article{})

    if q.Status != "" {
        query = query.Where("status = ?", q.Status)
    }

    var total int64
    if err := query.Count(&total).Error; err != nil {
        return nil, 0, err
    }

    var articles []Article
    err := query.
        Select("id", "title", "summary", "status", "created_at").
        Order("created_at DESC").
        Limit(q.PageSize).
        Offset((q.Page - 1) * q.PageSize).
        Find(&articles).Error

    return articles, total, err
}
```

评分标准：

- 页码和页大小校验：3 分。
- 动态条件：3 分。
- `Count` 和列表查询分开：3 分。
- 字段裁剪：3 分。
- 排序明确：2 分。
- 错误处理：2 分。

---

### 题目 7：软删除有什么好处和坑？

分值：10 分。

参考要点：

好处是可以恢复数据、保留审计记录、避免误删。坑是数据仍占存储，唯一索引仍生效，统计时要注意过滤条件，物理清理需要额外任务。

---

### 题目 8：请说明 Model、DTO、Repository、Service、Handler 的职责。

分值：20 分。

参考要点：

- Model：数据库结构。
- DTO：请求和响应结构。
- Repository：数据访问。
- Service：业务规则和事务。
- Handler：HTTP 参数解析和响应。

高分回答：

Service 不应该把 HTTP 细节传给 Repository，Repository 不应该决定事务边界，DTO 不应该直接复用数据库模型。

---

## 二、中级模拟面试卷

适用对象：学完关联、事务、性能排查、工程分层。

建议时长：60 分钟。

总分：100 分。

### 题目 1：请设计博客系统中用户、文章、分类、标签、评论的 GORM 模型。

分值：18 分。

参考实现：

```go
type User struct {
    ID        uint
    Username  string `gorm:"size:64;not null;uniqueIndex"`
    Articles  []Article
    Comments  []Comment
    CreatedAt time.Time
    UpdatedAt time.Time
}

type Category struct {
    ID       uint
    Name     string `gorm:"size:64;not null;uniqueIndex"`
    Articles []Article
}

type Tag struct {
    ID       uint
    Name     string `gorm:"size:64;not null;uniqueIndex"`
    Articles []Article `gorm:"many2many:article_tags;"`
}

type Article struct {
    ID         uint
    UserID     uint
    User       User
    CategoryID uint
    Category   Category
    Title      string `gorm:"size:200;not null;index"`
    Summary    string `gorm:"size:500"`
    Content    string `gorm:"type:text"`
    Status     string `gorm:"size:32;not null;index"`
    Tags       []Tag `gorm:"many2many:article_tags;"`
    Comments   []Comment
    CreatedAt  time.Time
    UpdatedAt  time.Time
}

type Comment struct {
    ID        uint
    ArticleID uint
    Article   Article
    UserID    uint
    User      User
    Content   string `gorm:"type:text;not null"`
    CreatedAt time.Time
}
```

评分标准：

- 关联关系正确：8 分。
- 索引意识：3 分。
- 字段类型合理：3 分。
- 不把所有字段都设计成字符串：2 分。
- 能解释多对多中间表：2 分。

---

### 题目 2：`Preload` 和 `Joins` 分别适合什么场景？

分值：10 分。

参考要点：

`Preload` 适合加载关联对象，尤其是一对多、多对多。`Joins` 适合按关联表字段筛选、排序或只需要平铺结果的查询。一对多 JOIN 容易造成主记录重复。

---

### 题目 3：请说明 N+1 查询是什么，并给出解决方案。

分值：10 分。

参考要点：

先查 1 次主列表，再循环 N 次查关联，叫 N+1。解决方案包括 `Preload`、JOIN、批量查询、缓存和调整接口结构。

---

### 题目 4：请实现创建文章并绑定标签的事务流程。

分值：18 分。

参考实现：

```go
func (s *ArticleService) CreateArticle(ctx context.Context, req CreateArticleRequest) error {
    return s.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        article := Article{
            UserID:     req.UserID,
            CategoryID: req.CategoryID,
            Title:      req.Title,
            Summary:    req.Summary,
            Content:    req.Content,
            Status:     "draft",
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

评分标准：

- 事务边界在 Service：3 分。
- 全部操作使用 `tx`：4 分。
- 标签存在性校验：3 分。
- 多对多关系处理正确：4 分。
- 错误返回触发回滚：2 分。
- 状态字段合理：2 分。

---

### 题目 5：请实现安全扣减库存。

分值：14 分。

参考实现：

```go
result := tx.Model(&Product{}).
    Where("id = ? AND stock >= ?", productID, quantity).
    UpdateColumn("stock", gorm.Expr("stock - ?", quantity))

if result.Error != nil {
    return result.Error
}
if result.RowsAffected == 0 {
    return ErrInsufficientStock
}
```

参考解释：

这个写法把库存判断和扣减放在一条 SQL 中，避免先查库存再扣减带来的并发超卖问题。

---

### 题目 6：如何排查一个 GORM 接口慢？

分值：12 分。

参考流程：

```text
1. 打开 SQL 日志
2. 看 SQL 条数和耗时
3. 判断是否 N+1
4. 判断是否查了大字段
5. 检查分页参数
6. 复制慢 SQL 执行 EXPLAIN
7. 看索引、扫描行数、排序、临时表
8. 根据结果优化查询、索引、缓存或接口设计
```

---

### 题目 7：GORM 如何避免 SQL 注入？

分值：8 分。

参考要点：

使用参数绑定，排序字段使用白名单，不把用户输入直接拼接到 `Where`、`Order`、`Table`、`Raw` 中。

---

### 题目 8：Repository 如何支持事务？

分值：10 分。

参考要点：

Repository 方法可以接收 `*gorm.DB`，或者 Repository 提供 `WithTx(tx)` 方法返回绑定事务的 Repository。事务边界由 Service 控制。

---

## 三、高级模拟面试卷

适用对象：准备中高级 Go 后端面试，已经完成博客系统和订单系统实战。

建议时长：90 分钟。

总分：100 分。

### 题目 1：你如何评价 ORM？什么时候用 GORM，什么时候直接写 SQL？

分值：12 分。

高分回答结构：

```text
1. ORM 是效率工具，不是 SQL 替代品
2. 普通 CRUD、分页、简单关联用 GORM
3. 复杂统计、窗口函数、CTE、性能瓶颈直接写 SQL
4. 所有关键查询都看 SQL 日志和执行计划
5. 项目中允许 GORM 和原生 SQL 共存
```

---

### 题目 2：请设计订单创建流程，并说明如何保证一致性。

分值：18 分。

答题要求：

- 说明订单主表、订单明细、商品库存。
- 说明事务边界。
- 说明库存并发扣减。
- 说明失败回滚。

参考流程：

```text
1. 校验用户和商品
2. 开启事务
3. 创建订单主表
4. 创建订单明细
5. 条件更新扣减库存
6. RowsAffected 为 0 则库存不足
7. 全部成功提交
8. 任一步失败回滚
```

---

### 题目 3：支付回调重复到达，如何保证不会重复更新订单？

分值：12 分。

参考要点：

- 支付流水号唯一索引。
- 订单状态机。
- 条件更新：只允许待支付订单变成已支付。
- 重复回调返回成功，但不重复发放权益。

参考实现：

```go
result := tx.Model(&Order{}).
    Where("id = ? AND status = ?", orderID, "pending").
    Updates(map[string]any{
        "status":  "paid",
        "paid_at": time.Now(),
    })

if result.Error != nil {
    return result.Error
}
if result.RowsAffected == 0 {
    return nil
}
```

---

### 题目 4：生产环境如何管理数据库结构变更？

分值：12 分。

高分回答：

本地可以用 `AutoMigrate`，生产使用 migration 文件。每次变更需要版本化、评审、备份和回滚预案。大表 DDL 要评估锁表风险，必要时分批、灰度或使用在线 DDL 方案。

---

### 题目 5：多租户系统中如何避免数据越权？

分值：10 分。

参考要点：

- 业务表必须包含 `tenant_id`。
- Repository 或 Scope 统一注入租户条件。
- 创建数据时写入租户 ID。
- 唯一索引可能要包含 `tenant_id`。
- 测试必须覆盖跨租户访问。

---

### 题目 6：请说明读写分离下的一致性问题。

分值：10 分。

参考要点：

读写分离可以提升读扩展能力，但主从复制有延迟。刚写完马上读、支付后查订单、权限变更后读取等强一致场景应该读主库，不能盲目走从库。

---

### 题目 7：你在 GORM 项目中遇到过哪些坑？

分值：12 分。

高分回答示例：

```text
我遇到过结构体 Updates 不更新零值的问题。
当时用户资料接口允许把昵称改为空字符串，但使用结构体 Updates 后 SQL 没有更新 nickname。
后来我查 GORM 文档和 SQL 日志，发现结构体更新会忽略零值。
最后改成 map 更新，并为这个场景补了测试。
```

评价标准：

- 有真实场景。
- 有定位过程。
- 有最终修复。
- 有复盘和测试。

---

### 题目 8：请把你的 GORM 项目讲成 3 分钟项目介绍。

分值：14 分。

回答模板：

```text
我做的是一个博客/订单系统，使用 Go、Gin、GORM 和 PostgreSQL/MySQL。
数据层用 GORM 建模用户、文章、分类、标签、评论/订单、订单明细、商品库存。
在 Repository 层封装查询，在 Service 层管理事务。
项目中比较关键的是两个点：
第一是关联关系，比如文章和标签是 many2many，文章详情用 Preload 加载分类、作者和标签，避免 N+1。
第二是一致性，比如创建订单时订单、订单明细和库存扣减放在同一个事务中，库存扣减使用条件更新和 RowsAffected 防止超卖。
性能方面，列表接口使用 Select 避免查询正文大字段，并为 status、category_id、created_at 建联合索引，慢查询通过 GORM 日志和 EXPLAIN 排查。
```

---

## 四、面试官追问库

### 基础追问

- 为什么不直接在 Handler 中写 GORM？
- `Find` 查不到数据会不会返回错误？
- `Delete` 一定是物理删除吗？
- 为什么列表接口不要查询正文？
- `Order(req.Sort)` 有什么风险？

### 关联追问

- `Preload("Tags")` 里的 `Tags` 是表名还是字段名？
- 多对多中间表如何命名？
- 一对多 JOIN 为什么可能导致重复行？
- 评论列表要不要和文章详情一起加载？

### 事务追问

- 事务里某一步用了外部 `db` 会怎样？
- 库存扣减为什么不能先查再扣？
- 乐观锁和悲观锁怎么选？
- 事务中可以调用外部服务吗？

### 性能追问

- 如何判断是不是 N+1？
- `EXPLAIN` 主要看什么？
- 大分页为什么慢？
- 连接池设置过大有什么风险？

### 工程追问

- `AutoMigrate` 能不能直接用于生产？
- Repository 返回什么错误更合适？
- 如何测试事务回滚？
- GORM 和 SQLC、Ent、原生 SQL 怎么取舍？

---

## 五、评分复盘表

每次模拟面试后，用这张表复盘：

| 项目 | 自评分 | 问题 |
| --- | ---: | --- |
| GORM API 熟练度 | 0-20 | 是否能写出关键代码 |
| SQL 理解 | 0-20 | 是否能解释底层 SQL |
| 事务一致性 | 0-20 | 是否能说清事务边界和回滚 |
| 性能排查 | 0-20 | 是否能说出日志、EXPLAIN、索引 |
| 项目表达 | 0-20 | 是否有真实场景和结果 |

达标标准：

- 80 分以上：可以正常参加 GORM 相关面试。
- 60 到 79 分：能面初中级岗位，但项目表达还要加强。
- 60 分以下：先回到阶段教程补代码练习。
