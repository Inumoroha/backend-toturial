# 07 Update、Updates、Save 与零值问题

本节目标：掌握 GORM 更新数据的几种方式，尤其是结构体零值不会被 `Updates` 默认更新的问题。这是 GORM 新手最高频的坑之一。

---

## 一、准备数据

```go
user := User{
    Username: "tom",
    Email:    "tom@example.com",
    Age:      20,
}

db.Create(&user)
```

---

## 二、Update 更新单个字段

```go
err := db.Model(&User{}).
    Where("id = ?", user.ID).
    Update("age", 21).Error
```

大致 SQL：

```sql
UPDATE users SET age = 21, updated_at = ...
WHERE id = ?;
```

适合只更新一个字段。

---

## 三、Updates 使用 map

```go
err := db.Model(&User{}).
    Where("id = ?", user.ID).
    Updates(map[string]any{
        "username": "tom_new",
        "age":      0,
    }).Error
```

使用 `map` 时，零值会被更新。

这里 `age = 0` 会真正写入数据库。

---

## 四、Updates 使用结构体

```go
err := db.Model(&User{}).
    Where("id = ?", user.ID).
    Updates(User{
        Username: "",
        Age:      0,
    }).Error
```

默认情况下，空字符串和 0 是零值，会被忽略。

也就是说，`username` 和 `age` 可能都不会更新。

---

## 五、强制更新结构体零值

使用 `Select`：

```go
err := db.Model(&User{}).
    Where("id = ?", user.ID).
    Select("username", "age").
    Updates(User{
        Username: "",
        Age:      0,
    }).Error
```

这样零值也会更新。

---

## 六、Save 的特点

```go
user.Age = 30
err := db.Save(&user).Error
```

`Save` 更像保存整个对象。它会根据主键判断更新还是创建。

风险：

- 容易更新你没打算改的字段。
- 如果结构体不是从数据库完整查出来的，可能把其他字段覆盖成零值。

真实项目中，更推荐明确使用 `Updates` 指定要更新的字段。

---

## 七、更新时检查 RowsAffected

```go
result := db.Model(&User{}).
    Where("id = ?", 99999).
    Update("age", 18)

if result.Error != nil {
    return result.Error
}

if result.RowsAffected == 0 {
    return ErrUserNotFound
}
```

SQL 执行成功不代表真的更新到了数据。不存在的 ID 会影响 0 行。

---

## 八、防止全表更新

GORM 默认会阻止没有条件的批量更新：

```go
db.Model(&User{}).Update("age", 18)
```

你应该明确写条件：

```go
db.Model(&User{}).Where("age < ?", 18).Update("age", 18)
```

---

## 九、常见错误

### 1. 用结构体更新零值

如果想把年龄更新成 0，请用 map 或 Select。

### 2. 滥用 Save

`Save` 对初学者很方便，但真实项目里要谨慎。更新接口最好只更新允许修改的字段。

### 3. 不检查 RowsAffected

编辑用户、编辑文章、删除评论时，都应该检查是否影响了行。

---

## 十、本节达标标准

学完本节后，你应该能够做到：

- 使用 `Update` 更新单字段。
- 使用 `Updates(map)` 更新多个字段。
- 解释结构体零值为什么没更新。
- 使用 `Select` 强制更新零值。
- 说明 `Save` 的风险。
- 使用 `RowsAffected` 判断数据是否存在。

---

## 颗粒度补强：本节应该如何真正练会

这一节不要只停留在“看懂概念”。建议你按下面的方式把「Update零值与Save差异」练成可以在项目中使用的能力。

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
