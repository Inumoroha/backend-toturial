# 08 AutoMigrate 与迁移边界

本节目标：理解 `AutoMigrate` 适合什么、不适合什么。它能帮助你学习和原型开发，但生产迁移需要更严谨的流程。

---

## 一、AutoMigrate 做什么

```go
db.AutoMigrate(&User{}, &Article{})
```

它会根据模型尝试创建或调整表结构。

学习阶段常用于：

- 快速创建表。
- 添加新字段。
- 验证模型 tag。
- 观察 GORM 生成的结构。

---

## 二、AutoMigrate 不是什么

它不是完整迁移系统。

不要指望它自动安全完成所有变更，比如：

- 删除字段。
- 重命名字段。
- 拆表。
- 合并表。
- 复杂索引调整。
- 数据迁移。

这些操作需要明确 SQL 脚本和发布流程。

---

## 三、学习阶段推荐用法

```go
func Migrate(db *gorm.DB) error {
    return db.AutoMigrate(
        &model.User{},
        &model.Category{},
        &model.Tag{},
        &model.Article{},
        &model.Comment{},
    )
}
```

放在：

```text
internal/database/migrate.go
```

启动时调用：

```go
if err := database.Migrate(db); err != nil {
    log.Fatal(err)
}
```

---

## 四、每次迁移后要检查

执行后在数据库里查看：

```sql
SHOW TABLES;
SHOW CREATE TABLE articles;
SHOW INDEX FROM articles;
```

不要只相信程序没报错。你要确认表结构符合预期。

---

## 五、生产环境迁移思路

生产更推荐：

```text
1. 写迁移 SQL
2. Code Review
3. 测试库执行
4. 备份数据
5. 灰度或低峰发布
6. 记录迁移版本
7. 准备回滚方案
```

可以使用迁移工具：

- golang-migrate
- goose
- atlas

GORM 模型仍然负责应用层映射，但表结构变更由迁移工具管理。

---

## 六、字段重命名示例

原字段：

```go
Name string
```

改成：

```go
Username string
```

`AutoMigrate` 可能会认为这是新增 `username` 字段，而不是把 `name` 改名。

正确生产做法可能是：

```sql
ALTER TABLE users CHANGE name username varchar(64) NOT NULL;
```

这类变更不能盲目交给自动迁移。

---

## 七、常见错误

### 1. 生产环境启动自动 AutoMigrate

这可能在没有审查的情况下改表，风险很高。

### 2. 以为删除结构体字段会删除数据库列

`AutoMigrate` 通常不会自动删除列。

### 3. 不检查索引是否创建

tag 写错可能导致索引没按预期创建。

---

## 八、本节达标标准

学完本节后，你应该能够做到：

- 使用 `AutoMigrate` 创建学习表。
- 说清 `AutoMigrate` 的边界。
- 在数据库中检查迁移结果。
- 知道生产环境为什么需要迁移工具。
- 解释字段重命名为什么不能盲目依赖自动迁移。

---

## 颗粒度补强：本节应该如何真正练会

这一节不要只停留在“看懂概念”。建议你按下面的方式把「AutoMigrate与迁移边界」练成可以在项目中使用的能力。

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
