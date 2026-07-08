# GORM 面试题与参考答案

本节目标：系统整理 Go 后端面试中和 GORM 相关的高频问题。你不只要知道 API 怎么用，还要能解释背后的 SQL、工程化取舍、事务一致性、性能优化和真实项目落地方式。

建议复习方式：

```text
1. 先自己口述答案
2. 再看参考答案补充细节
3. 把答案和博客系统、订单系统项目联系起来
4. 对每个问题准备一个真实代码例子
5. 面试时优先讲业务场景，再讲 GORM API
```

---

## 一、GORM 基础概念

### 1. 什么是 GORM？

参考答案：

GORM 是 Go 语言中常用的 ORM 库。ORM 的意思是 Object Relational Mapping，也就是对象关系映射。它可以把 Go 结构体映射到数据库表，把结构体字段映射到数据库列，让开发者用 Go 代码完成数据库的增删改查、关联查询、事务、迁移等操作。

例如：

```go
type User struct {
    ID    uint
    Name  string
    Email string
}
```

默认会映射到数据库中的 `users` 表。

面试中可以补充：

GORM 的价值不是让我们完全不用 SQL，而是减少重复 CRUD 代码，提高模型和业务代码之间的表达效率。真正用好 GORM，仍然要理解它生成的 SQL。

容易踩坑：

- 只会写 GORM API，但看不懂 SQL。
- 所有查询都强行用 GORM 链式 API，复杂统计反而变得难维护。
- 不理解 ORM 可能带来的 N+1 查询问题。

项目回答方式：

在博客系统中，我用 GORM 定义用户、文章、分类、标签、评论模型，并通过 `Preload` 处理文章详情关联查询；在订单系统中，我使用 GORM 事务保证订单、订单明细和库存扣减的一致性。

---

### 2. GORM 和原生 SQL 相比有什么优缺点？

参考答案：

GORM 的优点：

- 减少重复 CRUD 代码。
- 结构体和表结构映射清晰。
- 支持关联关系、事务、Hook、Scope、软删除。
- 和 Go 类型系统结合较好。
- 适合常见业务开发，提高开发效率。

GORM 的缺点：

- 生成 SQL 不一定总是最优。
- 不熟悉时容易写出 N+1 查询。
- 复杂 SQL 用链式 API 可能可读性差。
- 对数据库高级特性的表达不如原生 SQL 直接。
- 如果不看 SQL 日志，排查问题会困难。

什么时候用 GORM：

- 普通 CRUD。
- 简单条件查询。
- 分页查询。
- 常规关联查询。
- 中小型业务系统。

什么时候用原生 SQL：

- 复杂报表统计。
- 多层子查询。
- 窗口函数。
- CTE。
- 性能高度敏感的查询。
- 数据库特定语法。

面试表达：

我不会把 GORM 和 SQL 对立起来。日常业务优先用 GORM 提高效率，复杂查询或性能瓶颈场景会直接写原生 SQL，并通过 `EXPLAIN` 分析执行计划。

---

### 3. GORM 的默认命名约定是什么？

参考答案：

GORM 默认使用约定优于配置：

- 结构体名转为复数蛇形表名。
- 字段名转为蛇形列名。
- `ID` 字段默认是主键。
- `CreatedAt`、`UpdatedAt` 会被自动维护。
- 如果有 `gorm.DeletedAt` 字段，默认启用软删除。

示例：

```go
type BlogArticle struct {
    ID        uint
    UserID    uint
    CreatedAt time.Time
}
```

默认映射：

```text
表名：blog_articles
字段：id, user_id, created_at
```

自定义表名：

```go
func (User) TableName() string {
    return "app_users"
}
```

容易踩坑：

- `Preload` 使用的是结构体字段名，不是表名。
- Go 字段 `PasswordHash` 默认列名是 `password_hash`。
- 不理解默认复数表名，导致查错表。

---

### 4. GORM 中 `gorm.Model` 包含哪些字段？

参考答案：

`gorm.Model` 大致包含：

```go
type Model struct {
    ID        uint `gorm:"primarykey"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

使用：

```go
type User struct {
    gorm.Model
    Name string
}
```

优点：

- 少写重复字段。
- 自动拥有主键、创建时间、更新时间、软删除字段。

缺点：

- 字段不够显式。
- 不方便自定义 ID 类型。
- 不方便控制 JSON 输出。
- 初学时不利于理解表结构。

项目建议：

学习阶段可以用 `gorm.Model`，真实项目中我更倾向于显式写字段，这样模型含义更清楚，也更便于控制响应 DTO。

---

## 二、模型设计与字段映射

### 5. GORM 中如何定义主键？

参考答案：

默认情况下，字段名为 `ID` 会被 GORM 识别为主键：

```go
type User struct {
    ID uint
}
```

也可以显式声明：

```go
type User struct {
    ID uint `gorm:"primaryKey"`
}
```

如果使用非 ID 字段作为主键：

```go
type User struct {
    UUID string `gorm:"primaryKey;size:36"`
}
```

面试补充：

大多数业务表使用自增 ID 或雪花 ID、UUID 作为主键。自增 ID 简单高效；UUID 更适合分布式生成，但索引局部性和存储成本需要考虑。

---

### 6. GORM 中如何设置唯一索引和普通索引？

参考答案：

唯一索引：

```go
Email string `gorm:"size:128;not null;uniqueIndex"`
```

普通索引：

```go
UserID uint `gorm:"index"`
```

联合索引：

```go
CategoryID uint      `gorm:"index:idx_category_status_created"`
Status     string    `gorm:"index:idx_category_status_created"`
CreatedAt  time.Time `gorm:"index:idx_category_status_created"`
```

使用场景：

- 邮箱、用户名、订单号适合唯一索引。
- 外键字段适合普通索引。
- 高频组合查询适合联合索引。

项目例子：

文章列表常按 `category_id + status + created_at` 查询和排序，所以可以设计联合索引。

容易踩坑：

- 给所有字段乱加索引。
- 联合索引不根据查询设计。
- 只在业务代码判断唯一，没有数据库唯一约束。

---

### 7. GORM 中如何实现软删除？

参考答案：

模型中加入：

```go
DeletedAt gorm.DeletedAt `gorm:"index"`
```

执行：

```go
db.Delete(&user)
```

实际不是物理删除，而是更新 `deleted_at`：

```sql
UPDATE users SET deleted_at = ... WHERE id = ...
```

普通查询会自动加：

```sql
deleted_at IS NULL
```

查询包含软删除数据：

```go
db.Unscoped().Find(&users)
```

永久删除：

```go
db.Unscoped().Delete(&user)
```

适合软删除的表：

- 用户。
- 文章。
- 评论。
- 需要审计的数据。

不适合软删除的表：

- 临时表。
- 中间表。
- 高频日志表。
- 不需要保留历史的数据。

容易踩坑：

- 以为 `Delete` 一定是物理删除。
- 软删除数据仍然占用唯一索引。
- 使用 `Table()` 查询时忘记自己加 `deleted_at IS NULL`。

---

### 8. 软删除和唯一索引冲突怎么处理？

参考答案：

如果用户表有：

```go
Email string `gorm:"uniqueIndex"`
DeletedAt gorm.DeletedAt `gorm:"index"`
```

用户软删除后，数据仍然在表中，邮箱唯一索引仍然存在。如果再次注册同一个邮箱，可能会唯一冲突。

常见处理方式：

1. 不允许重复注册，提示账号已存在或已注销。
2. 支持账号恢复。
3. 物理删除确实不需要保留的账号。
4. 设计包含删除标记的联合唯一索引。
5. 删除时对唯一字段做脱敏改写，例如追加删除时间后缀。

面试建议：

不要简单说“用软删除就好了”。软删除会影响唯一约束、统计查询、数据清理和索引设计，必须结合业务决定。

---

### 9. GORM 中字段零值有什么坑？

参考答案：

使用结构体执行 `Updates` 时，GORM 默认不会更新零值字段：

```go
db.Model(&user).Updates(User{
    Name: "",
    Age:  0,
})
```

空字符串、0、false 都是 Go 的零值，默认会被忽略。

解决方式：

使用 map：

```go
db.Model(&user).Updates(map[string]any{
    "name": "",
    "age":  0,
})
```

或者使用 `Select`：

```go
db.Model(&user).
    Select("name", "age").
    Updates(User{Name: "", Age: 0})
```

面试重点：

这是 GORM 非常高频的坑。更新接口中如果允许把字段改成空字符串、0 或 false，一定要注意结构体更新的零值忽略行为。

---

### 10. Model 和 DTO 为什么要分离？

参考答案：

Model 是数据库结构，DTO 是接口输入输出结构，两者职责不同。

例如用户模型：

```go
type User struct {
    ID           uint
    Username     string
    Email        string
    PasswordHash string
}
```

如果直接返回 Model，可能暴露 `PasswordHash`。

响应 DTO：

```go
type UserResponse struct {
    ID       uint   `json:"id"`
    Username string `json:"username"`
}
```

分离好处：

- 避免敏感字段泄露。
- 控制接口字段。
- 避免前端请求直接影响数据库模型。
- 方便接口版本演进。
- 让数据库结构和 API 结构独立变化。

项目回答：

在博客系统中，我不会直接返回 `model.User`，而是转换成 `UserDTO`，避免返回密码哈希、软删除字段等内部字段。

---

## 三、CRUD 高频问题

### 11. `Create` 插入后主键会回填吗？

参考答案：

会。GORM 使用指针插入时，数据库生成的自增主键会回填到结构体。

```go
user := User{Name: "Tom"}
db.Create(&user)
fmt.Println(user.ID)
```

注意必须传指针：

```go
db.Create(&user)
```

不要传值：

```go
db.Create(user)
```

项目场景：

创建订单后，需要使用 `order.ID` 创建订单明细，所以主键回填非常重要。

---

### 12. GORM 如何批量插入？

参考答案：

```go
users := []User{
    {Name: "u1"},
    {Name: "u2"},
}

db.Create(&users)
```

指定批大小：

```go
db.CreateInBatches(users, 100)
```

适合：

- 批量导入。
- 初始化测试数据。
- 批量创建标签或商品。

注意：

- 批量过大可能导致 SQL 太长。
- 唯一索引冲突可能导致整批失败。
- 大批量导入要考虑事务、日志和数据库压力。

---

### 13. `First`、`Take`、`Last`、`Find` 有什么区别？

参考答案：

`First`：

- 按主键升序取第一条。
- 查不到返回 `gorm.ErrRecordNotFound`。

```go
db.First(&user)
```

`Last`：

- 按主键降序取第一条。
- 查不到返回 `gorm.ErrRecordNotFound`。

```go
db.Last(&user)
```

`Take`：

- 不指定排序取一条。
- 查不到返回 `gorm.ErrRecordNotFound`。

```go
db.Take(&user)
```

`Find`：

- 查询多条。
- 查不到通常返回空切片，不返回 `ErrRecordNotFound`。

```go
db.Find(&users)
```

面试重点：

详情查询通常用 `First` 并处理 `ErrRecordNotFound`；列表查询用 `Find` 并判断切片长度。

---

### 14. 如何处理 `record not found`？

参考答案：

```go
err := db.Where("email = ?", email).First(&user).Error
if err != nil {
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return ErrUserNotFound
    }
    return err
}
```

注意：

- `First`、`Take`、`Last` 查不到会返回 `ErrRecordNotFound`。
- `Find` 查不到多条数据一般返回空切片。

项目场景：

登录时邮箱不存在，不应该让服务崩溃，而是返回统一的“账号或密码错误”。

---

### 15. `Save` 和 `Updates` 有什么区别？

参考答案：

`Save` 会保存整个对象，如果主键存在则更新，否则创建。它可能更新所有字段。

```go
db.Save(&user)
```

`Updates` 用于更新指定字段：

```go
db.Model(&user).Updates(map[string]any{
    "name": "Tom",
    "age":  18,
})
```

区别：

- `Save` 更像全量保存。
- `Updates` 更适合局部更新。
- `Updates` 使用结构体时默认忽略零值。
- `Save` 如果对象字段不完整，可能误覆盖数据。

项目建议：

真实更新接口中，我更倾向于使用 `Updates(map)` 或 `Select(...).Updates(...)` 明确更新字段，避免 `Save` 误更新。

---

### 16. GORM 如何防止无条件全表更新？

参考答案：

GORM 默认会阻止没有条件的批量更新或删除，避免误操作。

错误示例：

```go
db.Model(&User{}).Update("age", 18)
```

正确写法：

```go
db.Model(&User{}).
    Where("id = ?", id).
    Update("age", 18)
```

如果确实要全表更新，需要显式允许，但生产中要非常谨慎。

面试表达：

更新和删除必须带明确条件，并检查 `RowsAffected`，尤其是后台批量操作。

---

### 17. `RowsAffected` 有什么用？

参考答案：

`RowsAffected` 表示 SQL 影响了多少行。

```go
result := db.Model(&User{}).
    Where("id = ?", id).
    Update("age", 18)

if result.Error != nil {
    return result.Error
}
if result.RowsAffected == 0 {
    return ErrUserNotFound
}
```

常见场景：

- 更新资源时判断资源是否存在。
- 删除资源时判断是否真的删除。
- 扣库存时判断库存是否足够。
- 支付订单时判断状态是否仍然是 pending。

面试重点：

SQL 执行成功不等于业务成功。比如库存不足时，更新语句可能不报错，但影响 0 行。

---

## 四、查询与分页

### 18. GORM 如何写条件查询？

参考答案：

```go
db.Where("email = ?", email).First(&user)
```

多个条件：

```go
db.Where("category_id = ?", categoryID).
    Where("status = ?", "published").
    Find(&articles)
```

不要拼接用户输入：

```go
db.Where("email = '" + email + "'")
```

因为有 SQL 注入风险。

---

### 19. 如何实现动态查询条件？

参考答案：

```go
query := db.Model(&Article{})

if categoryID > 0 {
    query = query.Where("category_id = ?", categoryID)
}

if keyword != "" {
    query = query.Where("title LIKE ?", "%"+keyword+"%")
}

if status != "" {
    query = query.Where("status = ?", status)
}

err := query.Find(&articles).Error
```

注意：

- 链式调用要重新赋值给 `query`。
- 用户输入要使用占位符。
- 排序字段要白名单。

项目场景：

文章列表接口通常支持分类、关键词、状态、分页等动态条件。

---

### 20. GORM 如何实现分页？

参考答案：

```go
page := 1
pageSize := 10
offset := (page - 1) * pageSize

db.Limit(pageSize).
    Offset(offset).
    Order("created_at DESC").
    Find(&articles)
```

分页参数要保护：

```go
if page < 1 {
    page = 1
}
if pageSize <= 0 || pageSize > 100 {
    pageSize = 10
}
```

返回 total：

```go
var total int64
db.Model(&Article{}).Where(...).Count(&total)
```

容易踩坑：

- 不限制 `page_size`。
- 没有稳定排序。
- 大 offset 性能差。
- Count 查询和列表查询条件不一致。

---

### 21. 大分页怎么优化？

参考答案：

传统 offset 分页：

```sql
LIMIT 20 OFFSET 100000
```

当 offset 很大时，数据库需要跳过大量数据，会变慢。

优化方式：

使用游标分页：

```go
query := db.Where("status = ?", "published")
if lastID > 0 {
    query = query.Where("id < ?", lastID)
}

query.Order("id DESC").Limit(pageSize).Find(&articles)
```

适合：

- 信息流。
- 消息列表。
- 加载更多。

不适合：

- 必须跳到第 N 页的后台管理表格。

---

### 22. 如何做排序？有什么安全问题？

参考答案：

固定排序：

```go
db.Order("created_at DESC").Find(&articles)
```

如果排序来自前端，不能直接：

```go
db.Order(req.Sort)
```

应该白名单：

```go
func resolveSort(sort string) string {
    switch sort {
    case "created_at_asc":
        return "created_at ASC"
    case "view_count_desc":
        return "view_count DESC"
    default:
        return "created_at DESC"
    }
}
```

原因：

`Order` 中直接使用用户输入可能有 SQL 注入风险。

---

### 23. 如何只查询部分字段？

参考答案：

```go
db.Select("id", "title", "summary", "created_at").
    Find(&articles)
```

适合：

- 列表页不查询正文。
- 用户信息不查询密码哈希。
- 减少网络传输。
- 减少扫描字段。

项目场景：

文章列表页只需要标题、摘要、作者、分类、时间，不应该查询 `content` 正文。

---

### 24. GORM 如何执行原生 SQL？

参考答案：

查询：

```go
db.Raw(`
    SELECT a.id, a.title, u.username
    FROM articles a
    JOIN users u ON u.id = a.user_id
    WHERE a.status = ?
`, "published").Scan(&items)
```

执行更新：

```go
db.Exec(`
    UPDATE articles
    SET view_count = view_count + 1
    WHERE id = ?
`, id)
```

注意：

- 仍然使用参数绑定。
- 不拼接用户输入。
- 复杂统计可以用原生 SQL。

---

### 25. `Scan` 和 `Find` 有什么区别？

参考答案：

`Find` 通常用于查询 GORM 模型：

```go
var users []User
db.Find(&users)
```

`Scan` 通常用于查询自定义结果结构：

```go
type ArticleStats struct {
    CategoryID uint
    Total      int64
}

db.Model(&Article{}).
    Select("category_id, COUNT(*) AS total").
    Group("category_id").
    Scan(&stats)
```

项目场景：

统计类、JOIN 后的扁平 DTO、报表查询通常用 `Scan`。

---

## 五、关联关系

### 26. GORM 支持哪些关联关系？

参考答案：

常见关联：

- `belongs to`：属于。
- `has one`：一对一。
- `has many`：一对多。
- `many2many`：多对多。

例子：

```text
Article belongs to User
User has many Articles
Article has many Comments
Article many2many Tags
```

---

### 27. belongs to 怎么定义？

参考答案：

文章属于用户：

```go
type Article struct {
    ID     uint
    UserID uint
    User   User
    Title  string
}
```

外键在当前表：

```text
articles.user_id
```

查询：

```go
db.Preload("User").First(&article, id)
```

---

### 28. has many 怎么定义？

参考答案：

用户拥有多篇文章：

```go
type User struct {
    ID       uint
    Articles []Article
}

type Article struct {
    ID     uint
    UserID uint
}
```

查询用户和文章：

```go
db.Preload("Articles").First(&user, id)
```

注意：

一对多关系中，外键通常放在“多”的一方。

---

### 29. many2many 怎么定义？

参考答案：

文章和标签：

```go
type Article struct {
    ID   uint
    Tags []Tag `gorm:"many2many:article_tags;"`
}

type Tag struct {
    ID       uint
    Articles []Article `gorm:"many2many:article_tags;"`
}
```

GORM 会使用中间表：

```text
article_tags
```

常见操作：

```go
db.Model(&article).Association("Tags").Append(&tag)
db.Model(&article).Association("Tags").Replace(&tags)
db.Model(&article).Association("Tags").Clear()
```

编辑文章标签通常用 `Replace`，不是 `Append`。

---

### 30. `Preload` 是什么？

参考答案：

`Preload` 是 GORM 用于预加载关联数据的方式。

```go
db.Preload("User").
    Preload("Category").
    Preload("Tags").
    First(&article, id)
```

它通常会执行多条 SQL：

```sql
SELECT * FROM articles WHERE id = ?;
SELECT * FROM users WHERE id IN (...);
SELECT * FROM categories WHERE id IN (...);
```

注意：

`Preload` 使用结构体字段名，不是表名。

错误：

```go
Preload("users")
```

正确：

```go
Preload("User")
```

---

### 31. 条件预加载怎么写？

参考答案：

只加载已通过审核评论：

```go
db.Preload("Comments", "status = ?", "approved").
    First(&article, id)
```

带排序：

```go
db.Preload("Comments", func(db *gorm.DB) *gorm.DB {
    return db.Order("created_at ASC")
}).First(&article, id)
```

嵌套预加载：

```go
db.Preload("Comments.User").First(&article, id)
```

---

### 32. `Preload` 和 `Joins` 有什么区别？

参考答案：

`Preload`：

- 查询主表后，再查询关联表。
- 适合加载嵌套结构。
- 常用于详情页。

`Joins`：

- 使用 SQL JOIN。
- 适合扁平 DTO。
- 适合按关联表字段筛选或排序。

示例：

文章详情：

```go
db.Preload("User").Preload("Tags").First(&article, id)
```

文章列表 DTO：

```go
db.Table("articles AS a").
    Select("a.id, a.title, u.username AS author_name").
    Joins("JOIN users AS u ON u.id = a.user_id").
    Scan(&items)
```

面试表达：

我通常在详情页使用 `Preload`，在列表页需要扁平展示或按关联字段筛选时使用 `Joins`。

---

### 33. 什么是 N+1 查询？如何解决？

参考答案：

N+1 查询是指先查出 N 条主记录，再循环查询每条记录的关联数据。

错误示例：

```go
var articles []Article
db.Find(&articles)

for i := range articles {
    db.First(&articles[i].User, articles[i].UserID)
}
```

如果有 100 篇文章，就会执行 101 条 SQL。

解决：

```go
db.Preload("User").Find(&articles)
```

或者使用 JOIN 查询 DTO：

```go
db.Table("articles AS a").
    Joins("JOIN users AS u ON u.id = a.user_id").
    Scan(&items)
```

排查方式：

- 开启 SQL 日志。
- 看是否有大量重复查询。
- 看 SQL 条数是否随记录数线性增长。

---

## 六、事务与一致性

### 34. GORM 如何使用事务？

参考答案：

推荐闭包事务：

```go
err := db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&order).Error; err != nil {
        return err
    }

    if err := tx.Create(&items).Error; err != nil {
        return err
    }

    return nil
})
```

规则：

- 返回 nil 提交。
- 返回 error 回滚。
- 事务内部必须使用 `tx`。

手动事务：

```go
tx := db.Begin()
if err := tx.Create(&order).Error; err != nil {
    tx.Rollback()
    return err
}
return tx.Commit().Error
```

---

### 35. 事务内部为什么不能使用外部 `db`？

参考答案：

因为外部 `db` 不在当前事务上下文中。

错误：

```go
db.Transaction(func(tx *gorm.DB) error {
    tx.Create(&order)
    db.Create(&orderItem)
    return nil
})
```

`orderItem` 这一步没有进入事务。如果后续回滚，可能只回滚 `order`，不回滚 `orderItem`。

正确：

```go
tx.Create(&orderItem)
```

面试重点：

事务内部所有数据库操作必须使用 `tx`，这是 GORM 事务最重要的注意点之一。

---

### 36. 下单为什么需要事务？

参考答案：

下单通常涉及多步写入：

```text
1. 创建订单
2. 创建订单明细
3. 扣减库存
```

这些步骤必须一起成功或一起失败。如果创建订单成功但扣库存失败，就会出现脏数据。

GORM 实现：

```go
db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&order).Error; err != nil {
        return err
    }
    if err := tx.Create(&items).Error; err != nil {
        return err
    }
    if err := reduceStock(tx, items); err != nil {
        return err
    }
    return nil
})
```

---

### 37. 如何安全扣减库存？

参考答案：

不要只先查库存再扣：

```go
if product.Stock >= quantity {
    product.Stock -= quantity
    db.Save(&product)
}
```

并发下可能超卖。

推荐条件更新：

```go
result := tx.Model(&Product{}).
    Where("id = ? AND stock >= ?", productID, quantity).
    Update("stock", gorm.Expr("stock - ?", quantity))

if result.Error != nil {
    return result.Error
}
if result.RowsAffected == 0 {
    return ErrStockNotEnough
}
```

重点：

- 库存判断放在 UPDATE 条件里。
- 检查 `RowsAffected`。
- 放在事务中。

---

### 38. 悲观锁怎么写？

参考答案：

使用 `clause.Locking`：

```go
err := tx.Clauses(clause.Locking{Strength: "UPDATE"}).
    First(&product, productID).Error
```

生成类似：

```sql
SELECT * FROM products WHERE id = ? FOR UPDATE;
```

注意：

- 必须在事务中使用。
- 锁持有时间要短。
- 不要在锁内做外部网络请求。

适合：

- 强一致。
- 冲突较高。
- 需要串行修改同一资源。

---

### 39. 乐观锁怎么实现？

参考答案：

增加版本号：

```go
type Product struct {
    ID      uint
    Stock   int
    Version int
}
```

更新时带版本条件：

```go
result := db.Model(&Product{}).
    Where("id = ? AND version = ?", product.ID, product.Version).
    Updates(map[string]any{
        "stock":   gorm.Expr("stock - ?", quantity),
        "version": gorm.Expr("version + 1"),
    })
```

如果 `RowsAffected == 0`，说明版本冲突或库存不足。

适合：

- 冲突较少。
- 可重试。
- 读多写少。

---

### 40. GORM 默认事务是什么？可以关闭吗？

参考答案：

GORM 默认会对写操作启用事务，以保证数据一致性。但这会有一定性能开销。

关闭：

```go
db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
    SkipDefaultTransaction: true,
})
```

是否关闭要看场景：

适合考虑关闭：

- 高性能批量写入。
- 单表简单写入。
- 外层已经显式事务。

不建议关闭：

- 初学阶段。
- 订单、库存、支付等一致性场景。
- 关联创建。

---

## 七、性能优化

### 41. GORM 如何打印 SQL？

参考答案：

单次查询：

```go
db.Debug().Where("id = ?", id).First(&user)
```

全局 logger：

```go
db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
    Logger: logger.Default.LogMode(logger.Info),
})
```

自定义慢查询阈值：

```go
logger.New(
    log.New(os.Stdout, "\r\n", log.LstdFlags),
    logger.Config{
        SlowThreshold: time.Second,
        LogLevel:      logger.Warn,
    },
)
```

面试表达：

性能排查时，我会先打开 SQL 日志，看 SQL 条数、是否有 N+1、是否查询了多余字段，再用 EXPLAIN 看执行计划。

---

### 42. 如何排查 GORM 慢查询？

参考答案：

排查流程：

```text
1. 确认哪个接口慢
2. 打开 SQL 日志
3. 统计 SQL 条数
4. 找出慢 SQL
5. 执行 EXPLAIN
6. 判断是否索引缺失、N+1、大分页、字段过多
7. 优化后再次验证
```

常见原因：

- 没有分页。
- 查询了大字段。
- N+1 查询。
- 缺少索引。
- 大 offset。
- 统计实时全表扫描。

---

### 43. 如何使用 EXPLAIN？

参考答案：

复制 GORM 生成的 SQL：

```sql
EXPLAIN
SELECT id, title
FROM articles
WHERE status = 'published'
ORDER BY created_at DESC
LIMIT 10;
```

重点看：

- `key`：实际使用的索引。
- `rows`：预计扫描行数。
- `Extra`：是否有 filesort、temporary。

优化流程：

- 根据 WHERE 和 ORDER BY 设计索引。
- 再次 EXPLAIN 验证。

---

### 44. GORM 如何配置连接池？

参考答案：

GORM 底层使用 `database/sql`，可以这样配置：

```go
sqlDB, err := db.DB()
if err != nil {
    return err
}

sqlDB.SetMaxOpenConns(100)
sqlDB.SetMaxIdleConns(10)
sqlDB.SetConnMaxLifetime(time.Hour)
sqlDB.SetConnMaxIdleTime(10 * time.Minute)
```

参数含义：

- `MaxOpenConns`：最大打开连接数。
- `MaxIdleConns`：最大空闲连接数。
- `ConnMaxLifetime`：连接最大生命周期。
- `ConnMaxIdleTime`：连接最大空闲时间。

面试补充：

连接池参数要结合数据库最大连接数、服务实例数、接口并发量和 SQL 平均耗时来调整。

---

### 45. 如何避免列表页查询大字段？

参考答案：

使用 `Select`：

```go
db.Model(&Article{}).
    Select("id", "title", "summary", "created_at").
    Where("status = ?", "published").
    Find(&articles)
```

文章正文 `content` 通常只在详情页查询，不应该在列表页返回。

项目表达：

博客系统文章列表只返回标题、摘要、作者、分类、时间，不返回正文；详情页再加载正文和关联数据。

---

### 46. GORM 批量操作有哪些注意点？

参考答案：

批量插入：

```go
db.CreateInBatches(users, 100)
```

批量更新：

```go
db.Model(&Article{}).
    Where("status = ?", "draft").
    Update("status", "published")
```

注意：

- 批量更新必须带条件。
- 大批量写入可能需要分批。
- 关注事务大小。
- 关注唯一索引冲突。
- 关注数据库写入压力。

---

## 八、高级特性

### 47. GORM Hook 是什么？适合做什么？

参考答案：

Hook 是模型生命周期钩子，例如：

- `BeforeCreate`
- `AfterCreate`
- `BeforeUpdate`
- `AfterUpdate`
- `BeforeDelete`
- `AfterFind`

示例：

```go
func (u *User) BeforeCreate(tx *gorm.DB) error {
    if u.UUID == "" {
        u.UUID = uuid.NewString()
    }
    return nil
}
```

适合：

- 自动生成 UUID。
- 填充简单默认值。
- 模型内部简单校验。

不适合：

- 复杂业务流程。
- 调外部服务。
- 发消息。
- 权限判断。
- 大事务。

面试表达：

我会把复杂业务放在 Service 层，Hook 只做轻量、稳定、和模型强相关的事情。

---

### 48. Scope 是什么？

参考答案：

Scope 是可复用查询条件，类型是：

```go
func(*gorm.DB) *gorm.DB
```

分页 Scope：

```go
func Paginate(page, pageSize int) func(*gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        return db.Offset((page - 1) * pageSize).Limit(pageSize)
    }
}
```

使用：

```go
db.Scopes(Paginate(1, 10)).Find(&articles)
```

适合：

- 分页。
- 状态过滤。
- 租户过滤。
- 通用查询条件。

不适合：

- 复杂业务流程。
- 很长的报表查询。

---

### 49. GORM 如何使用 Context？

参考答案：

```go
db.WithContext(ctx).First(&user, id)
```

Repository 标准写法：

```go
func (r *UserRepository) FindByID(ctx context.Context, id uint) (*User, error) {
    var user User
    err := r.db.WithContext(ctx).First(&user, id).Error
    return &user, err
}
```

用途：

- 请求取消。
- 超时控制。
- 链路追踪。
- 日志字段传递。

Handler 中传递：

```go
service.FindByID(c.Request.Context(), id)
```

不要在 Repository 里随便用 `context.Background()`。

---

### 50. GORM 如何保存 JSON 字段？

参考答案：

可以使用 serializer：

```go
type ArticleMeta struct {
    CoverURL string   `json:"cover_url"`
    Keywords []string `json:"keywords"`
}

type Article struct {
    ID   uint
    Meta ArticleMeta `gorm:"serializer:json"`
}
```

适合：

- 扩展信息。
- 不常筛选的附加字段。
- 用户偏好设置。

不适合：

- 标签关系。
- 订单明细。
- 高频查询字段。

---

### 51. GORM Gen 是什么？什么时候用？

参考答案：

GORM Gen 是 GORM 的代码生成工具，可以生成类型安全查询代码。

普通 GORM：

```go
db.Where("email = ?", email)
```

Gen：

```go
u := query.User
u.Where(u.Email.Eq(email)).First()
```

优点：

- 减少字符串字段名错误。
- 更类型安全。
- 适合中大型项目。

不适合：

- 刚入门。
- 小 demo。
- 模型频繁变化的早期项目。

---

### 52. GORM 支持读写分离吗？

参考答案：

GORM 可以通过 Database Resolver 插件支持多数据源和读写分离。

基本思路：

- 写操作走主库。
- 读操作走从库。
- 事务通常走主库。

注意：

读写分离会带来复制延迟问题。比如写完立刻读，如果读走从库，可能读不到刚写入的数据。

面试表达：

读写分离不是简单配置就完事，需要考虑一致性、延迟、事务边界和故障切换。

---

## 九、工程化设计

### 53. GORM 在项目中应该放在哪一层？

参考答案：

推荐分层：

```text
Handler    处理 HTTP 请求和响应
Service    处理业务规则、事务、权限
Repository 处理 GORM 数据库访问
Model      数据库表映射
DTO        请求和响应结构
```

GORM 操作通常放在 Repository 层。

Service 层组织事务：

```go
db.Transaction(func(tx *gorm.DB) error {
    articleRepo := repo.WithDB(tx)
    return articleRepo.Create(...)
})
```

Handler 不应该直接 `db.Create`。

---

### 54. Repository 层的作用是什么？

参考答案：

Repository 封装数据库访问，让业务层不直接关心 GORM 细节。

职责：

- 创建、查询、更新、删除。
- 封装复杂查询。
- 封装关联预加载。
- 提供事务中的 `WithDB(tx)` 能力。

不负责：

- HTTP 状态码。
- 权限判断。
- JWT 解析。
- 业务状态流转。

---

### 55. Service 层应该做什么？

参考答案：

Service 负责业务规则：

- 参数业务校验。
- 权限判断。
- 状态流转。
- 多 Repository 协作。
- 事务边界。
- 返回业务错误。

示例：

创建文章时：

```text
1. 校验标题
2. 校验分类存在
3. 校验标签存在
4. 开启事务
5. 创建文章
6. 绑定标签
```

这些都属于 Service。

---

### 56. 如何让 Repository 支持事务？

参考答案：

可以给 Repository 增加：

```go
func (r *ArticleRepository) WithDB(db *gorm.DB) *ArticleRepository {
    return &ArticleRepository{db: db}
}
```

Service 中：

```go
err := s.db.Transaction(func(tx *gorm.DB) error {
    articleRepo := s.articleRepo.WithDB(tx)
    return articleRepo.Create(ctx, article)
})
```

这样 Repository 可以复用同样的方法，只是底层 DB 换成事务对象 `tx`。

---

### 57. 如何做 GORM 测试？

参考答案：

分层测试：

Repository 测试：

- 查询条件。
- 分页。
- 关联加载。
- 软删除。

Service 测试：

- 业务规则。
- 权限。
- 状态流转。
- 事务回滚。

Handler 测试：

- 请求参数。
- 状态码。
- 响应格式。

建议使用独立测试库：

```text
gorm_study_test
```

不要用开发库跑测试。

---

## 十、项目场景题

### 58. 如何设计博客系统的 GORM 模型？

参考答案：

核心模型：

- User
- Article
- Category
- Tag
- Comment

关系：

```text
User has many Articles
User has many Comments
Article belongs to User
Article belongs to Category
Article has many Comments
Article many2many Tags
Comment belongs to User
Comment belongs to Article
```

关键点：

- 用户邮箱唯一。
- 文章有软删除。
- 评论有软删除。
- 文章和标签多对多。
- 文章列表按 `category_id + status + created_at` 建索引。
- 详情页使用 `Preload` 加载关联。
- 创建文章和绑定标签放在事务中。

---

### 59. 创建文章并绑定标签怎么保证一致性？

参考答案：

流程：

```text
1. 校验分类存在
2. 校验标签存在
3. 开启事务
4. 创建文章
5. Replace 标签关联
6. 提交事务
```

GORM：

```go
db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&article).Error; err != nil {
        return err
    }
    if err := tx.Model(&article).Association("Tags").Replace(&tags); err != nil {
        return err
    }
    return nil
})
```

如果标签绑定失败，文章创建也要回滚。

---

### 60. 如何设计文章列表接口？

参考答案：

支持：

- 分页。
- 分类筛选。
- 关键词搜索。
- 状态筛选。
- 排序。

注意：

- 不查询正文。
- 返回 total。
- 排序字段白名单。
- 分类、状态、创建时间设计索引。

查询可以用 JOIN DTO：

```go
db.Table("articles AS a").
    Select("a.id, a.title, a.summary, u.username AS author_name, c.name AS category_name").
    Joins("JOIN users AS u ON u.id = a.user_id").
    Joins("JOIN categories AS c ON c.id = a.category_id").
    Where("a.status = ?", "published").
    Limit(pageSize).
    Offset(offset).
    Scan(&items)
```

---

### 61. 如何设计文章详情接口？

参考答案：

文章详情需要：

- 作者。
- 分类。
- 标签。
- 评论。
- 评论用户。

GORM：

```go
db.Preload("User").
    Preload("Category").
    Preload("Tags").
    Preload("Comments.User").
    First(&article, id)
```

注意：

- 评论很多时要拆成评论分页接口。
- 不要在循环中查询评论用户。
- 使用 DTO 避免暴露敏感字段。

---

### 62. 如何设计订单系统下单流程？

参考答案：

模型：

- Product
- Order
- OrderItem

流程：

```text
1. 校验订单项
2. 查询商品
3. 计算金额
4. 创建订单
5. 创建订单明细
6. 条件更新扣库存
7. 提交事务
```

关键：

```go
Where("id = ? AND stock >= ?", productID, quantity).
Update("stock", gorm.Expr("stock - ?", quantity))
```

必须检查 `RowsAffected`。

---

### 63. 如何防止重复支付？

参考答案：

基础方案：

- 订单状态必须是 `pending`。
- 支付单号唯一。
- 更新订单状态时带状态条件。
- 检查 `RowsAffected`。
- 支付回调做幂等。

GORM：

```go
result := tx.Model(&Order{}).
    Where("id = ? AND status = ?", orderID, "pending").
    Update("status", "paid")

if result.RowsAffected == 0 {
    return ErrOrderAlreadyPaidOrInvalid
}
```

---

### 64. 订单取消如何恢复库存？

参考答案：

流程：

```text
1. 查询订单和明细
2. 判断订单状态是 pending
3. 修改订单状态 cancelled
4. 遍历订单明细恢复库存
5. 使用事务
```

恢复库存：

```go
tx.Model(&Product{}).
    Where("id = ?", item.ProductID).
    Update("stock", gorm.Expr("stock + ?", item.Quantity))
```

注意：

已支付订单不能取消。

---

## 十一、开放性面试题

### 65. 你在项目中遇到过哪些 GORM 坑？

参考回答方向：

可以讲这些：

1. 结构体 Updates 不更新零值。
2. Preload 字段名写错。
3. Find 查不到不返回 ErrRecordNotFound。
4. 软删除数据仍然占用唯一索引。
5. N+1 查询。
6. 事务内部误用外部 db。
7. 列表页查询了大字段。
8. 没检查 RowsAffected。

示例回答：

我遇到过结构体更新零值的问题。比如用户把年龄改成 0，用 `Updates(User{Age:0})` 时 GORM 默认忽略零值，导致数据库没有变化。后来我改成 `Updates(map[string]any{"age":0})`，或者使用 `Select("age")` 明确更新字段。

---

### 66. 如果一个 GORM 接口很慢，你怎么排查？

参考答案：

我会按顺序排查：

```text
1. 打开 SQL 日志
2. 看 SQL 条数，判断是否 N+1
3. 看是否查询了大字段
4. 看是否分页
5. 复制慢 SQL 执行 EXPLAIN
6. 检查索引是否命中
7. 判断是否需要优化查询、加索引、拆接口或加缓存
8. 优化后再次压测或对比日志
```

项目例子：

文章列表慢时，我会先确认是否查询了 `content` 正文，再看是否有 `LIMIT`，然后检查 `category_id/status/created_at` 索引。

---

### 67. 你如何评价 ORM？

参考答案：

ORM 是提高开发效率的工具，不是 SQL 的替代品。对于普通 CRUD、分页、简单关联，ORM 可以减少重复代码，让模型和业务表达更清晰。但对于复杂统计、性能敏感查询、数据库特定能力，原生 SQL 更直接。

我使用 ORM 的原则是：

```text
能清晰表达的普通业务用 GORM
复杂统计和性能瓶颈用原生 SQL
所有关键查询都要看生成 SQL
```

---

### 68. GORM 项目如何做代码组织？

参考答案：

我通常这样分层：

```text
model       GORM 模型
repository  GORM 数据访问
service     业务规则和事务
handler     HTTP 请求响应
dto         请求和响应结构
database    DB 初始化和迁移
config      配置
```

这样做的好处：

- Handler 不直接操作数据库。
- Service 可以统一管理事务。
- Repository 可以复用查询。
- DTO 避免暴露数据库模型。
- 测试更容易分层编写。

---

### 69. 如何把 GORM 项目讲成简历亮点？

参考答案：

不要写：

```text
使用 GORM 完成增删改查。
```

太普通。

可以写：

```text
基于 Go、Gin、GORM、MySQL 实现博客系统，使用 GORM 建模用户、文章、分类、标签和评论关系，通过 many2many 处理中间表，使用 Preload 优化文章详情关联查询，使用事务保证文章创建和标签绑定的一致性，并通过分页、字段裁剪和联合索引优化文章列表查询。
```

订单系统：

```text
基于 GORM 事务实现订单创建、订单明细写入和库存扣减，通过条件更新和 RowsAffected 判断避免库存超扣，使用订单状态机和唯一支付单号处理重复支付问题，并编写事务回滚测试覆盖库存不足场景。
```

---

## 十二、补充高频追问

### 70. `AutoMigrate` 在生产环境中可以直接用吗？

参考答案：

`AutoMigrate` 可以根据 GORM 模型自动创建表、补充缺失字段、补充索引和约束，但它不适合作为所有生产数据库变更的唯一方案。

示例：

```go
err := db.AutoMigrate(&User{}, &Article{}, &Comment{})
```

学习阶段可以用它快速建表，项目早期也可以用来降低初始化成本。但生产环境更推荐使用可审计、可回滚的迁移工具，例如 goose、golang-migrate，或者由团队统一维护 SQL migration。

原因：

- 生产数据库变更需要明确知道执行了哪些 DDL。
- 表字段删除、字段类型调整、数据迁移通常不能只靠自动迁移完成。
- 大表 DDL 可能锁表，必须评估执行窗口和影响。
- 多人协作时，迁移顺序需要被版本化管理。

面试表达：

我会在本地开发和测试环境使用 `AutoMigrate` 提升效率，但生产环境会把结构变更写成 migration 文件，经过评审、备份、灰度和回滚预案后再执行。GORM 模型负责表达业务结构，migration 负责管理数据库演进。

---

### 71. GORM 的 `*gorm.DB` 是线程安全的吗？

参考答案：

通常情况下，初始化后的 `*gorm.DB` 可以作为全局依赖复用，它内部基于 `database/sql` 连接池，不需要每次请求都重新打开数据库连接。

常见写法：

```go
var DB *gorm.DB

func InitDB() {
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        panic(err)
    }
    DB = db
}
```

需要注意：

- `gorm.Open` 不是每个请求调用一次，而是应用启动时调用一次。
- 每个请求可以基于全局 `DB` 创建查询链。
- 事务对象 `tx` 只在当前事务流程中使用，不要保存为全局变量。
- 如果用 `Session` 修改局部配置，应该明确它的作用范围。

容易踩坑：

- 在每个 Handler 中 `gorm.Open`，导致连接暴增。
- 把事务 `tx` 传到异步 goroutine 后继续使用，事务生命周期变得不可控。
- 没有配置连接池，压测时出现连接耗尽。

---

### 72. `gorm.Open` 和底层 `database/sql` 是什么关系？

参考答案：

GORM 是建立在 Go 标准库 `database/sql` 之上的 ORM。`gorm.Open` 会创建 GORM 的数据库对象，而真正的连接池管理能力来自底层的 `*sql.DB`。

获取底层连接池：

```go
sqlDB, err := db.DB()
if err != nil {
    return err
}

sqlDB.SetMaxOpenConns(50)
sqlDB.SetMaxIdleConns(10)
sqlDB.SetConnMaxLifetime(time.Hour)
```

面试重点：

GORM 不是自己维护所有 TCP 连接，它会复用标准库连接池。因此连接池参数、连接生命周期、空闲连接数、最大打开连接数仍然是后端工程师必须理解的内容。

---

### 73. 什么是 `Session`？什么时候会用到？

参考答案：

`Session` 用来为当前操作创建一个带特殊配置的 GORM 会话，不影响全局 `db`。

示例：

```go
tx := db.Session(&gorm.Session{
    DryRun: true,
})

stmt := tx.Where("email = ?", email).Find(&User{}).Statement
fmt.Println(stmt.SQL.String())
fmt.Println(stmt.Vars)
```

常见用途：

- `DryRun`：只生成 SQL，不真正执行。
- `PrepareStmt`：当前会话启用预编译缓存。
- `SkipDefaultTransaction`：当前会话跳过默认事务。
- `NewDB`：创建一个不带之前查询条件的新会话。
- `FullSaveAssociations`：完整保存关联数据。

面试补充：

`Session` 的价值是把配置限制在一次操作或一段局部逻辑里，避免全局配置被误改。

---

### 74. `DryRun` 有什么用？

参考答案：

`DryRun` 可以让 GORM 只生成 SQL 和参数，不真正访问数据库，适合调试复杂查询和写测试时检查 SQL 形状。

示例：

```go
stmt := db.Session(&gorm.Session{DryRun: true}).
    Where("status = ?", "published").
    Order("created_at DESC").
    Limit(10).
    Find(&[]Article{}).Statement

fmt.Println(stmt.SQL.String())
fmt.Println(stmt.Vars)
```

输出类似：

```text
SELECT * FROM "articles" WHERE status = $1 ORDER BY created_at DESC LIMIT 10
[published]
```

注意：

`DryRun` 只能帮助你观察 SQL 结构，不能代替真实数据库执行计划分析。真正的性能判断仍然要在数据库中执行 `EXPLAIN` 或 `EXPLAIN ANALYZE`。

---

### 75. `Unscoped` 是什么？

参考答案：

如果模型中包含 `gorm.DeletedAt`，GORM 默认启用软删除。普通查询会自动过滤已删除数据：

```sql
WHERE deleted_at IS NULL
```

使用 `Unscoped` 可以查询或物理删除软删除数据。

查询包含已删除数据：

```go
var users []User
db.Unscoped().Find(&users)
```

物理删除：

```go
db.Unscoped().Delete(&user)
```

面试重点：

`Unscoped` 权限很大，应该谨慎使用。后台恢复数据、管理员审计、数据清理任务可能会用到它，但普通业务接口不应该随意暴露物理删除能力。

---

### 76. `Select` 和 `Omit` 有什么区别？

参考答案：

`Select` 表示只处理指定字段，`Omit` 表示排除指定字段。它们可以用于查询、创建和更新。

只查询部分字段：

```go
db.Select("id", "title", "created_at").Find(&articles)
```

创建时排除字段：

```go
db.Omit("Role").Create(&user)
```

更新零值字段：

```go
db.Model(&user).
    Select("nickname", "age").
    Updates(User{Nickname: "", Age: 0})
```

面试建议：

列表接口应该优先使用 `Select` 控制字段，避免把正文、大 JSON、长文本字段查出来。更新接口如果允许写入零值，也可以用 `Select` 明确告诉 GORM 哪些字段必须更新。

---

### 77. GORM 中如何避免 SQL 注入？

参考答案：

核心原则是：用户输入必须通过参数绑定传入，不要拼接到 SQL 字符串中。

安全写法：

```go
db.Where("email = ?", req.Email).First(&user)
```

危险写法：

```go
db.Where("email = '" + req.Email + "'").First(&user)
```

排序字段尤其容易被忽略：

```go
// 危险：sort 来自用户输入
db.Order(req.Sort).Find(&users)
```

安全方式是白名单：

```go
allowedSorts := map[string]string{
    "created_desc": "created_at DESC",
    "created_asc":  "created_at ASC",
}

order, ok := allowedSorts[req.Sort]
if !ok {
    order = "created_at DESC"
}

db.Order(order).Find(&users)
```

面试重点：

`Where("field = ?", value)` 这种参数绑定是安全的；`Order`、`Table`、`Select`、`Raw` 中如果拼接用户输入，就需要格外小心，优先使用白名单。

---

### 78. `FirstOrCreate`、`Attrs`、`Assign` 有什么区别？

参考答案：

`FirstOrCreate` 会先按条件查询，查不到时创建。

```go
db.Where(User{Email: email}).FirstOrCreate(&user)
```

`Attrs` 只在创建时设置默认值，查到已有数据时不会更新：

```go
db.Where(User{Email: email}).
    Attrs(User{Name: "new user"}).
    FirstOrCreate(&user)
```

`Assign` 无论查到还是创建，都会把字段赋值到结果对象；配合 `FirstOrCreate` 时，查到已有记录也可能触发更新语义，需要谨慎确认行为。

```go
db.Where(User{Email: email}).
    Assign(User{LastLoginAt: time.Now()}).
    FirstOrCreate(&user)
```

容易踩坑：

- 以为 `Attrs` 会更新已有记录。
- 使用 `FirstOrCreate` 做高并发唯一数据创建时，没有数据库唯一索引兜底。
- 没有处理并发下的唯一冲突。

面试建议：

高并发场景不能只依赖“先查再插”的应用逻辑，唯一约束必须落在数据库层。

---

### 79. GORM 如何处理数据库唯一冲突？

参考答案：

唯一冲突应该优先通过数据库唯一索引保证，然后在应用层捕获错误并转换成业务错误。

示例模型：

```go
type User struct {
    ID    uint
    Email string `gorm:"uniqueIndex;size:128"`
}
```

创建：

```go
err := db.Create(&user).Error
if err != nil {
    // 根据驱动错误码判断是否唯一冲突
    return ErrEmailAlreadyExists
}
```

为什么不能只先查再插：

```text
请求 A 查询：邮箱不存在
请求 B 查询：邮箱不存在
请求 A 插入成功
请求 B 插入冲突
```

面试表达：

应用层检查可以提升用户提示体验，但不能代替数据库唯一索引。真正防止重复数据的最后防线应该是数据库约束。

---

### 80. GORM 的关联保存要注意什么？

参考答案：

创建主对象时，如果结构体中带有关联对象，GORM 可能会自动保存关联。这个能力很方便，但也容易让写入行为变得不透明。

示例：

```go
user := User{
    Name: "Tom",
    Profile: Profile{
        Bio: "hello",
    },
}

db.Create(&user)
```

如果不希望保存某些关联，可以使用 `Omit`：

```go
db.Omit("Profile").Create(&user)
```

如果希望完整保存关联，可以考虑：

```go
db.Session(&gorm.Session{FullSaveAssociations: true}).Updates(&user)
```

面试重点：

真实项目中，我更倾向于把复杂关联写入拆成明确的步骤，并放在事务里完成。这样更容易处理错误、回滚和业务校验。

---

### 81. Association Mode 是什么？

参考答案：

Association Mode 是 GORM 提供的关联操作接口，可以对某个模型的关联数据执行追加、替换、删除、清空、计数等操作。

示例：文章绑定标签：

```go
err := db.Model(&article).
    Association("Tags").
    Replace(tags)
```

常见方法：

- `Append`：追加关联。
- `Replace`：替换关联。
- `Delete`：删除指定关联关系。
- `Clear`：清空关联关系。
- `Count`：统计关联数量。

容易踩坑：

- Association 名称使用结构体字段名，比如 `Tags`，不是表名 `tags`。
- 多对多的 `Delete` 通常删除的是关联关系，不一定删除标签实体。
- 批量替换关联时要考虑事务。

项目表达：

在文章编辑接口中，我会用事务更新文章主体，再用 `Association("Tags").Replace(tags)` 替换标签关系，保证文章内容和标签绑定一致提交或一起回滚。

---

### 82. 外键约束要不要交给 GORM 自动创建？

参考答案：

GORM 可以通过模型关联和 tag 创建外键约束，例如：

```go
type OrderItem struct {
    ID      uint
    OrderID uint
    Order   Order `gorm:"constraint:OnUpdate:CASCADE,OnDelete:RESTRICT;"`
}
```

是否自动创建要看团队规范：

- 学习阶段可以让 GORM 创建，便于理解关联。
- 生产环境更推荐用 migration 明确维护外键和索引。
- 高并发大表是否使用外键，需要结合业务、性能、运维能力决定。

面试表达：

我会区分“逻辑关联”和“数据库外键约束”。GORM 模型一定要表达清楚关联关系，但数据库层是否强制外键，要结合系统规模、数据一致性要求和团队 DBA 规范决定。

---

### 83. `PrepareStmt` 有什么作用？

参考答案：

`PrepareStmt` 会缓存预编译语句，减少重复解析 SQL 的成本。

全局启用：

```go
db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
    PrepareStmt: true,
})
```

局部启用：

```go
db.Session(&gorm.Session{PrepareStmt: true}).Find(&users)
```

注意：

- 不是所有场景都需要打开。
- SQL 形状变化很多时，缓存收益有限。
- 需要关注数据库和驱动层面的 prepared statement 行为。
- 长期运行服务要注意语句缓存带来的资源占用。

面试建议：

不要把 `PrepareStmt` 当成万能性能优化。性能问题应先通过 SQL 日志、慢查询和执行计划定位，再决定是否启用。

---

### 84. 如何自定义 GORM 的命名策略？

参考答案：

GORM 默认使用蛇形复数命名。如果项目已有数据库命名规范，可以通过 `NamingStrategy` 自定义。

示例：

```go
db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
    NamingStrategy: schema.NamingStrategy{
        TablePrefix:   "app_",
        SingularTable: true,
    },
})
```

效果：

```text
User -> app_user
ArticleCategory -> app_article_category
```

注意：

命名策略属于全局配置，项目开始前就应该确定。中途改变命名策略可能导致查错表、迁移混乱或线上事故。

---

### 85. GORM 的 Logger 应该怎么配置？

参考答案：

开发环境可以打印详细 SQL，生产环境应该控制日志级别并记录慢 SQL。

示例：

```go
newLogger := logger.New(
    log.New(os.Stdout, "\r\n", log.LstdFlags),
    logger.Config{
        SlowThreshold:             200 * time.Millisecond,
        LogLevel:                  logger.Warn,
        IgnoreRecordNotFoundError: true,
        Colorful:                  false,
    },
)

db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
    Logger: newLogger,
})
```

面试表达：

我会在开发环境打开 SQL 日志帮助学习和调试；生产环境重点记录慢 SQL、错误 SQL、请求 trace id 和耗时，避免把所有 SQL 打满日志系统。

---

### 86. 使用 `Rows` 或 `ScanRows` 时有什么注意点？

参考答案：

当查询结果很多，或者需要流式处理时，可以使用 `Rows`。

示例：

```go
rows, err := db.Model(&User{}).Rows()
if err != nil {
    return err
}
defer rows.Close()

for rows.Next() {
    var user User
    if err := db.ScanRows(rows, &user); err != nil {
        return err
    }
    // process user
}
```

注意：

- 必须 `defer rows.Close()`，否则可能导致连接不能及时归还连接池。
- 循环中要处理扫描错误。
- 大批量处理时要控制事务、锁和内存占用。

面试重点：

普通列表查询用 `Find` 就够了；超大结果集、导出任务、批处理任务才会考虑 `Rows` 或分批查询。

---

### 87. `FindInBatches` 适合什么场景？

参考答案：

`FindInBatches` 用于分批处理大量数据，避免一次性把所有记录加载到内存。

示例：

```go
err := db.Where("status = ?", "pending").
    FindInBatches(&orders, 500, func(tx *gorm.DB, batch int) error {
        for _, order := range orders {
            // process order
        }
        return nil
    }).Error
```

适用场景：

- 批量修复数据。
- 离线任务。
- 数据导出。
- 批量状态同步。

容易踩坑：

- 批处理过程中没有清空或复用切片时，要关注内存。
- 每批内部如果有写操作，要考虑事务边界。
- 数据在处理过程中变化时，要考虑是否需要固定快照或按 ID 游标推进。

---

### 88. GORM 如何支持 JSON 字段查询？

参考答案：

保存 JSON 字段可以使用 `datatypes.JSON`、自定义类型或数据库原生 JSON 类型。查询 JSON 字段时，通常会结合数据库特定语法。

PostgreSQL 示例：

```go
db.Where("profile ->> 'city' = ?", "Shanghai").Find(&users)
```

MySQL 示例：

```go
db.Where("JSON_EXTRACT(profile, '$.city') = ?", "Shanghai").Find(&users)
```

面试重点：

JSON 字段适合存储结构灵活、查询频率不高的扩展属性。如果字段经常参与筛选、排序、聚合，就应该考虑拆成普通列，或者为 JSON 表达式建立合适索引。

---

### 89. GORM 的泛型 API 是什么？

参考答案：

较新的 GORM 版本提供了泛型 API，可以让查询返回类型更明确，减少一些反射式写法的心智负担。

示例：

```go
user, err := gorm.G[User](db).Where("id = ?", id).First(ctx)
```

传统写法：

```go
var user User
err := db.WithContext(ctx).Where("id = ?", id).First(&user).Error
```

面试表达：

我了解 GORM 传统 API 和泛型 API。实际项目中会根据团队使用的 GORM 版本和代码风格选择，核心仍然是写出清晰 SQL、处理错误和管理事务。

---

### 90. 面试中被问“你真的用过 GORM 吗”怎么回答？

参考答案：

不要只说“我用它做过增删改查”。更好的回答方式是把业务、API、SQL 和问题排查串起来。

示例回答：

```text
我在博客系统里用 GORM 建了用户、文章、分类、标签和评论模型。
文章列表接口没有直接查正文，而是用 Select 只取 id、title、摘要、状态和创建时间。
文章详情用 Preload 加载作者、分类、标签和评论，并通过 SQL 日志检查是否出现 N+1。
创建文章时，文章主体和标签绑定放在同一个事务里，标签关系使用 Association("Tags").Replace 处理。
后来列表接口变慢，我通过 GORM 日志定位 SQL，再用 EXPLAIN 检查索引，最后加了 category_id、status、created_at 的联合索引。
```

这个答案的好处：

- 有项目背景。
- 有具体 GORM API。
- 有事务和关联。
- 有性能排查。
- 有数据库索引意识。

面试官通常会继续追问 `Preload`、事务、索引和慢查询，这些都可以接回前面的题目继续展开。

---

## 十三、速记清单

### 必须背熟的问题

- GORM 是什么？
- GORM 默认命名约定是什么？
- `First`、`Take`、`Find` 区别是什么？
- `Save` 和 `Updates` 区别是什么？
- 结构体更新零值为什么不生效？
- 如何软删除？
- 如何分页？
- 如何处理关联？
- `Preload` 和 `Joins` 区别是什么？
- 什么是 N+1？
- 如何使用事务？
- 事务内部为什么必须用 `tx`？
- 如何安全扣库存？
- 如何打印 SQL？
- 如何排查慢查询？
- 如何配置连接池？
- Model 和 DTO 为什么分离？
- Repository 和 Service 分别做什么？

### 面试回答原则

```text
先讲业务场景
再讲 GORM API
再讲生成 SQL
再讲常见坑
最后讲项目里怎么用
```

这样回答会比单纯背 API 更像真正做过项目。
