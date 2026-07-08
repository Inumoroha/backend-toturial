# 05 Belongs To 属于关系

本节目标：掌握 GORM 中最常见的 `belongs to` 关系。比如文章属于用户、文章属于分类、订单属于用户。

---

## 一、什么是 belongs to

`belongs to` 表示当前模型保存了另一个模型的外键。

典型例子：

```text
一篇文章属于一个用户。
```

数据库里表现为：

```text
articles.user_id -> users.id
```

也就是说，外键在 `articles` 表里。

---

## 二、模型写法

```go
type User struct {
    ID       uint
    Username string
}

type Article struct {
    ID     uint
    UserID uint `gorm:"not null;index"`
    User   User
    Title  string
}
```

关键字段：

```go
UserID uint
User   User
```

GORM 根据 `User` 推断外键是 `UserID`。

---

## 三、创建文章

真实项目中，创建文章时通常直接指定外键：

```go
article := Article{
    UserID: userID,
    Title:  "GORM belongs to",
}

err := db.Create(&article).Error
```

不需要每次都把完整 `User` 对象塞进去。

---

## 四、查询文章并加载作者

```go
var article Article

err := db.Preload("User").
    First(&article, articleID).Error
```

使用：

```go
fmt.Println(article.User.Username)
```

---

## 五、文章属于分类

```go
type Category struct {
    ID   uint
    Name string
}

type Article struct {
    ID         uint
    CategoryID uint `gorm:"not null;index"`
    Category   Category
    Title      string
}
```

查询：

```go
db.Preload("Category").First(&article, id)
```

---

## 六、自定义外键

如果你的字段不叫 `UserID`，可以显式配置：

```go
type Article struct {
    ID       uint
    AuthorID uint
    Author   User `gorm:"foreignKey:AuthorID"`
}
```

初学阶段建议尽量遵循默认约定，这样配置最少。

---

## 七、常见错误

### 1. 只有 User，没有 UserID

如果没有外键字段，创建和查询都会变得不清晰。

### 2. Preload 用表名

错误：

```go
Preload("users")
```

正确：

```go
Preload("User")
```

### 3. 创建文章时相信前端传 UserID

作者 ID 应该来自登录态，不应该由前端请求体决定。

---

## 八、本节达标标准

学完本节后，你应该能够做到：

- 解释 belongs to 的含义。
- 写出文章属于用户的模型。
- 写出文章属于分类的模型。
- 使用 `Preload` 加载 belongs to 数据。
- 知道什么时候需要 `foreignKey`。

---

## 颗粒度补强：本节应该如何真正练会

这一节不要只停留在“看懂概念”。建议你按下面的方式把「BelongsTo属于关系」练成可以在项目中使用的能力。

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
