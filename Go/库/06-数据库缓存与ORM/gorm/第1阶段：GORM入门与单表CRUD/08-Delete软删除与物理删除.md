# 08 Delete、软删除与物理删除

本节目标：掌握 GORM 删除数据的两种方式：物理删除和软删除。你要知道什么时候数据真的没了，什么时候只是被标记删除。

---

## 一、没有 DeletedAt 时是物理删除

模型：

```go
type Category struct {
    ID   uint
    Name string
}
```

删除：

```go
db.Delete(&Category{}, 1)
```

大致 SQL：

```sql
DELETE FROM categories WHERE id = 1;
```

数据会从表中消失。

---

## 二、有 DeletedAt 时是软删除

模型：

```go
type User struct {
    ID        uint
    Email     string
    DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

删除：

```go
db.Delete(&user)
```

大致 SQL：

```sql
UPDATE users SET deleted_at = ... WHERE id = ?;
```

数据仍然在表里，只是 `deleted_at` 不为空。

---

## 三、普通查询会自动过滤软删除

```go
var users []User
db.Find(&users)
```

GORM 会自动加条件：

```sql
WHERE deleted_at IS NULL
```

所以你用 GORM 查不到已软删除的数据。

---

## 四、查询包含软删除的数据

```go
var users []User
db.Unscoped().Find(&users)
```

`Unscoped` 表示不使用默认软删除范围。

只查询已删除数据：

```go
db.Unscoped().
    Where("deleted_at IS NOT NULL").
    Find(&users)
```

---

## 五、永久删除

```go
db.Unscoped().Delete(&user)
```

这会执行真正的 `DELETE`。

真实项目中要谨慎使用，通常只用于：

- 测试数据清理。
- 后台定时清理过期数据。
- 用户明确要求永久删除且业务允许。

---

## 六、批量删除

```go
result := db.Where("status = ?", "draft").Delete(&Article{})
```

一定要带条件。不要写无条件删除。

检查：

```go
if result.Error != nil {
    return result.Error
}
fmt.Println(result.RowsAffected)
```

---

## 七、删除前的业务判断

比如删除评论：

```text
1. 查询评论是否存在
2. 判断当前用户是否是评论作者
3. 执行软删除
```

不要只凭前端传来的 ID 直接删。

---

## 八、常见错误

### 1. 以为 Delete 一定物理删除

只要模型有 `gorm.DeletedAt`，普通 `Delete` 就是软删除。

### 2. 软删除后唯一索引冲突

如果邮箱唯一，软删除用户后再次注册同邮箱，仍然可能冲突，因为旧记录还在表里。

这类业务要单独设计，比如：

- 不允许重复注册。
- 恢复旧账号。
- 唯一索引中加入删除标记。

### 3. 列表统计包含软删除数据

使用 `Table` 写原生表查询时，GORM 不一定自动加软删除条件。要自己写：

```go
Where("deleted_at IS NULL")
```

---

## 九、本节达标标准

学完本节后，你应该能够做到：

- 区分物理删除和软删除。
- 知道 `gorm.DeletedAt` 的作用。
- 使用 `Unscoped` 查询软删除数据。
- 使用 `Unscoped().Delete` 永久删除。
- 删除时检查 `RowsAffected`。
- 理解软删除和唯一索引的冲突问题。

---

## 颗粒度补强：本节应该如何真正练会

这一节不要只停留在“看懂概念”。建议你按下面的方式把「Delete软删除与物理删除」练成可以在项目中使用的能力。

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
