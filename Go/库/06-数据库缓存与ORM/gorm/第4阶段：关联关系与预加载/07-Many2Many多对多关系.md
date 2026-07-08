# 07 Many2Many 多对多关系

本节目标：掌握 GORM 多对多关系。最典型的例子是文章和标签：一篇文章有多个标签，一个标签也属于多篇文章。

---

## 一、为什么需要中间表

多对多不能只靠一个外键表达。

错误想法：

```text
articles.tag_id
```

这只能表示一篇文章一个标签。

正确做法：

```text
articles
tags
article_tags
```

中间表保存：

```text
article_id
tag_id
```

---

## 二、模型写法

```go
type Article struct {
    ID    uint
    Title string
    Tags  []Tag `gorm:"many2many:article_tags;"`
}

type Tag struct {
    ID       uint
    Name     string
    Articles []Article `gorm:"many2many:article_tags;"`
}
```

关键是：

```go
gorm:"many2many:article_tags;"
```

---

## 三、迁移后检查

```go
db.AutoMigrate(&Article{}, &Tag{})
```

检查：

```sql
SHOW TABLES;
SHOW CREATE TABLE article_tags;
```

确认中间表存在。

---

## 四、创建文章并绑定标签

```go
var tags []Tag
db.Where("id IN ?", []uint{1, 2}).Find(&tags)

article := Article{
    Title: "GORM 多对多",
    Tags:  tags,
}

db.Create(&article)
```

GORM 会创建文章，并写入 `article_tags`。

---

## 五、给已有文章追加标签

```go
db.Model(&article).Association("Tags").Append(&tag)
```

`Append` 是追加，不会移除旧标签。

---

## 六、替换标签

```go
db.Model(&article).Association("Tags").Replace(&tags)
```

编辑文章时更常用 `Replace`，因为前端提交的是最终标签集合。

---

## 七、清空标签

```go
db.Model(&article).Association("Tags").Clear()
```

这会清空中间表关系，不会删除标签本身。

---

## 八、常见错误

### 1. 误以为删除关联会删除标签

删除 `article_tags` 关系，不等于删除 `tags` 表记录。

### 2. 编辑文章时用了 Append

旧标签不会移除，标签会越加越多。

### 3. 没校验标签是否存在

前端传来的 tag ID 必须校验。

---

## 九、本节达标标准

学完本节后，你应该能够做到：

- 解释为什么多对多需要中间表。
- 写出文章和标签的 many2many 模型。
- 创建文章并绑定标签。
- 使用 Append、Replace、Clear。
- 知道关联操作通常要放进事务。

---

## 颗粒度补强：本节应该如何真正练会

这一节不要只停留在“看懂概念”。建议你按下面的方式把「Many2Many多对多关系」练成可以在项目中使用的能力。

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
