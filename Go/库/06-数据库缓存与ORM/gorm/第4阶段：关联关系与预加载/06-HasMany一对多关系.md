# 06 Has Many 一对多关系

本节目标：掌握 GORM 的一对多关系。比如用户拥有多篇文章、文章拥有多条评论、订单拥有多个订单明细。

---

## 一、什么是 has many

```text
一个用户可以有多篇文章。
一篇文章只属于一个用户。
```

数据库里外键在“多”的一方：

```text
articles.user_id
```

模型里“一个用户拥有多篇文章”写成：

```go
Articles []Article
```

---

## 二、模型写法

```go
type User struct {
    ID       uint
    Username string
    Articles []Article
}

type Article struct {
    ID     uint
    UserID uint `gorm:"not null;index"`
    Title  string
}
```

---

## 三、查询用户和文章

```go
var user User

err := db.Preload("Articles").
    First(&user, userID).Error
```

使用：

```go
for _, article := range user.Articles {
    fmt.Println(article.Title)
}
```

---

## 四、条件预加载

只加载已发布文章：

```go
err := db.Preload("Articles", "status = ?", "published").
    First(&user, userID).Error
```

带排序：

```go
err := db.Preload("Articles", func(db *gorm.DB) *gorm.DB {
    return db.Where("status = ?", "published").
        Order("created_at DESC")
}).First(&user, userID).Error
```

---

## 五、文章和评论

```go
type Article struct {
    ID       uint
    Title    string
    Comments []Comment
}

type Comment struct {
    ID        uint
    ArticleID uint `gorm:"not null;index"`
    Content   string
}
```

查询文章详情：

```go
db.Preload("Comments").First(&article, articleID)
```

如果评论很多，不建议一次加载全部，应该单独做评论分页接口。

---

## 六、订单和订单明细

```go
type Order struct {
    ID    uint
    Items []OrderItem
}

type OrderItem struct {
    ID      uint
    OrderID uint `gorm:"not null;index"`
}
```

这是订单系统中最重要的一对多关系。

---

## 七、常见错误

### 1. 外键放错地方

一对多关系中，外键通常放在“多”的一方。

### 2. 列表页加载太多 has many

用户列表不要 Preload 每个用户全部文章。文章列表也不要加载全部评论。

### 3. 条件预加载忘记排序

如果展示最近文章，应该明确 `Order`。

---

## 八、本节达标标准

学完本节后，你应该能够做到：

- 写出用户和文章的一对多模型。
- 写出文章和评论的一对多模型。
- 使用 `Preload` 加载 has many。
- 使用条件预加载和排序。
- 判断什么时候不应该一次加载全部子数据。

---

## 颗粒度补强：本节应该如何真正练会

这一节不要只停留在“看懂概念”。建议你按下面的方式把「HasMany一对多关系」练成可以在项目中使用的能力。

### 1. 先写最小可运行示例

在 `code/gorm-study` 中准备一个最小 Go 程序，只保留和本节主题直接相关的模型、数据库连接和 GORM 调用。不要一开始就放进完整项目结构，否则你很难判断问题来自 GORM、分层、路由还是参数绑定。

建议流程：

```text
1. 定义最小模型
2. AutoMigrate 建表
3. 插入 2 到 3 条测试数据
4. 执行本节核心 GORM API
5. 打印 SQL 日志
6. 到数据库客户端里验证结果
```

### 2. 必须观察 SQL

学习 GORM 的关键不是背 API，而是看它生成的 SQL。执行本节代码时，建议临时开启：

```go
db = db.Debug()
```

或者在初始化时使用：

```go
Logger: logger.Default.LogMode(logger.Info)
```

你至少要回答：

```text
这段 GORM 代码生成了几条 SQL？
SQL 中有没有 WHERE 条件？
有没有 LIMIT 或 ORDER BY？
有没有使用软删除条件 deleted_at IS NULL？
更新或删除时 RowsAffected 是多少？
```

### 3. 用数据库客户端验证

不要只相信 Go 程序输出。每次运行后，都在 MySQL 客户端中执行对应查询，例如：

```sql
SHOW TABLES;
SHOW CREATE TABLE users;
SHOW INDEX FROM users;
SELECT * FROM users ORDER BY id DESC LIMIT 10;
```

如果本节涉及文章、评论、订单或标签，就去查对应表：

```sql
SELECT * FROM articles ORDER BY id DESC LIMIT 10;
SELECT * FROM comments ORDER BY id DESC LIMIT 10;
SELECT * FROM orders ORDER BY id DESC LIMIT 10;
SELECT * FROM article_tags ORDER BY article_id DESC LIMIT 10;
```

### 4. 主动制造一个错误

每节都建议故意制造一个小错误，然后观察报错和数据变化。比如：

```text
把数据库密码写错，观察连接错误。
把唯一字段重复插入，观察唯一索引错误。
把查询 ID 改成不存在的值，观察 ErrRecordNotFound。
把更新条件去掉，观察 GORM 是否阻止全表更新。
把事务里的 return err 去掉，观察是否错误提交。
```

你真正排查过错误，才算把这个知识点学稳。

### 5. 放回真实项目场景

最后把本节知识放回博客系统或订单系统中思考：

```text
博客系统里哪个接口会用到这个知识点？
订单系统里哪个流程会因为这个知识点写错而出问题？
这个知识点应该放在 Model、Repository、Service 还是 Handler？
它需要测试吗？需要看 SQL 日志吗？需要事务吗？
```

如果你能回答这些问题，说明你已经从“知道 API”进入到“能做后端项目”的层次。

### 6. 本节复盘问题

学完后请写下这几个答案：

```text
1. 本节最核心的 GORM API 是什么？
2. 它生成的 SQL 大致是什么？
3. 它最容易踩的坑是什么？
4. 如果不用 GORM，原生 SQL 会怎么写？
5. 在博客系统或订单系统中，它会出现在哪个模块？
```
