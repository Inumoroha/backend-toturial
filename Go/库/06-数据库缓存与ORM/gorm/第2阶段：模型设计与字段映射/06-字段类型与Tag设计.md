# 06 字段类型与 Tag 设计

本节目标：掌握 GORM tag 的常见写法，知道字符串、正文、金额、状态、布尔值、时间等字段应该如何设计。

---

## 一、GORM tag 的作用

```go
Email string `gorm:"size:128;not null;uniqueIndex"`
```

tag 不是给 Go 编译器看的，而是给 GORM 看的。它影响：

- 数据库字段类型。
- 是否允许为空。
- 默认值。
- 索引。
- 唯一约束。
- 字段读写行为。

---

## 二、字符串字段

用户名：

```go
Username string `gorm:"size:64;not null;uniqueIndex"`
```

邮箱：

```go
Email string `gorm:"size:128;not null;uniqueIndex"`
```

标题：

```go
Title string `gorm:"size:200;not null"`
```

原则：

- 有明确长度上限的字符串，用 `size`。
- 需要唯一的字段加 `uniqueIndex`。
- 必填字段加 `not null`。

---

## 三、长文本字段

文章正文：

```go
Content string `gorm:"type:text;not null"`
```

评论内容：

```go
Content string `gorm:"type:text;not null"`
```

不要用 `size:255` 存正文。正文长度不可控，应该用 `text`。

---

## 四、金额字段

不推荐：

```go
Price float64
```

推荐用整数分：

```go
Price int64 `gorm:"not null"`
```

例如：

```text
9900 表示 99.00 元
```

原因：浮点数有精度问题，金额不应该用浮点数表示。

---

## 五、状态字段

```go
Status string `gorm:"size:32;not null;default:draft;index"`
```

常见状态：

```text
draft
published
archived
```

状态字段通常会出现在查询条件里，所以经常需要索引。

---

## 六、布尔字段

```go
IsPublished bool `gorm:"not null;default:false"`
```

如果状态只有两种，可以用 bool。如果状态以后可能扩展，建议用 string 状态字段。

例如文章状态可能有：

```text
draft
published
archived
reviewing
rejected
```

这种情况用 `Status string` 更合适。

---

## 七、默认值

```go
ViewCount int `gorm:"not null;default:0"`
```

默认值适合：

- 阅读量。
- 库存初始值。
- 状态初始值。
- 布尔开关。

注意：默认值是数据库层面的兜底，业务代码里也应该明确设置关键字段。

---

## 八、忽略字段

```go
Token string `gorm:"-"`
```

这个字段不会映射到数据库。

适合：

- 临时计算字段。
- 接口返回字段。
- 不想持久化的业务状态。

但真实项目更推荐 DTO，不要把太多接口字段塞进 Model。

---

## 九、常见错误

### 1. 所有 string 都不写长度

这样数据库字段可能不符合预期，也不利于约束业务。

### 2. 金额用 float64

订单系统里这是严重设计问题。

### 3. 状态字段不加约束

至少要在 Service 层限制状态枚举。数据库层也可以进一步用检查约束，但 GORM 跨数据库表达会复杂一些。

---

## 十、本节达标标准

学完本节后，你应该能够做到：

- 为用户名、邮箱、标题设计合适的 tag。
- 为正文使用 `text`。
- 用整数表示金额。
- 在状态字段上设置默认值和索引。
- 判断 bool 和 string 状态字段的取舍。
- 使用 `gorm:"-"` 忽略字段。

---

## 颗粒度补强：本节应该如何真正练会

这一节不要只停留在“看懂概念”。建议你按下面的方式把「字段类型与Tag设计」练成可以在项目中使用的能力。

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
