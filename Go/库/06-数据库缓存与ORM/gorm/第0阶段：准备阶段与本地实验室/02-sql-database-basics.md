# 02 SQL 与数据库基础

本节目标：围绕「sql database basics」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

GORM 可以帮你少写很多重复 SQL，但它不会替你理解数据库。一个合格的 Go 后端工程师，必须知道 GORM 最终会生成什么 SQL，以及这些 SQL 对数据库有什么影响。

## 1. 表、行、列

数据库表可以理解为结构化数据集合。

例如用户表：

```sql
CREATE TABLE users (
    id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(64) NOT NULL,
    email VARCHAR(128) NOT NULL UNIQUE,
    age INT NOT NULL DEFAULT 18,
    created_at DATETIME,
    updated_at DATETIME
);
```

对应到 Go 结构体大致是：

```go
type User struct {
    ID        uint
    Name      string
    Email     string
    Age       int
    CreatedAt time.Time
    UpdatedAt time.Time
}
```

GORM 的核心工作之一，就是在结构体和数据库表之间做映射。

## 2. 基础 CRUD SQL

### 新增

```sql
INSERT INTO users (name, email, age)
VALUES ('Tom', 'tom@example.com', 20);
```

### 查询

```sql
SELECT id, name, email, age
FROM users
WHERE id = 1;
```

### 更新

```sql
UPDATE users
SET age = 21
WHERE id = 1;
```

### 删除

```sql
DELETE FROM users
WHERE id = 1;
```

学 GORM 时，你要经常问自己：这一行 GORM 代码会生成哪条 SQL？

## 3. 条件查询

常见条件：

```sql
SELECT * FROM users WHERE age >= 18;
SELECT * FROM users WHERE name LIKE '%tom%';
SELECT * FROM users WHERE id IN (1, 2, 3);
SELECT * FROM users WHERE email IS NOT NULL;
```

这些条件在 GORM 里通常用 `Where` 表达。

## 4. 排序与分页

```sql
SELECT *
FROM articles
ORDER BY created_at DESC
LIMIT 10 OFFSET 20;
```

含义：

- `ORDER BY created_at DESC`：按创建时间倒序。
- `LIMIT 10`：最多返回 10 条。
- `OFFSET 20`：跳过前 20 条。

分页是后端接口里最常见的查询能力之一。

## 5. 聚合查询

```sql
SELECT category_id, COUNT(*) AS total
FROM articles
GROUP BY category_id;
```

用于统计每个分类下有多少文章。

常见聚合函数：

- `COUNT`
- `SUM`
- `AVG`
- `MIN`
- `MAX`

## 6. 主键、外键、索引

### 主键

主键用于唯一标识一行数据。常见字段是 `id`。

```sql
id BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT
```

### 外键

外键用于表达表之间的关系，例如文章属于某个用户：

```sql
user_id BIGINT UNSIGNED NOT NULL
```

从业务角度看，`articles.user_id` 指向 `users.id`。

### 索引

索引用于加速查询。

```sql
CREATE INDEX idx_articles_user_id ON articles(user_id);
CREATE INDEX idx_articles_created_at ON articles(created_at);
```

常见适合加索引的字段：

- 经常出现在 `WHERE` 中的字段
- 经常用于排序的字段
- 外键字段
- 唯一约束字段

## 7. 事务

事务用于保证一组操作要么全部成功，要么全部失败。

典型例子：下单。

```text
1. 创建订单
2. 创建订单明细
3. 扣减库存
```

如果第 3 步失败，前两步也应该撤销，否则系统就会出现脏数据。

## 8. 连接池

后端服务不会每次请求都重新创建数据库连接，而是维护一组可复用连接。

你需要理解这些概念：

- 最大打开连接数
- 最大空闲连接数
- 连接最大存活时间
- 连接最大空闲时间

GORM 底层使用 Go 标准库的 `database/sql` 连接池。

## 本节练习

在数据库客户端中手写 SQL 完成：

- [ ] 创建 `users` 表。
- [ ] 插入 3 个用户。
- [ ] 查询年龄大于 18 的用户。
- [ ] 按创建时间倒序查询用户。
- [ ] 修改某个用户邮箱。
- [ ] 删除某个用户。
- [ ] 给 `email` 字段创建唯一索引。

## 检查点

学完本节，你应该能解释：

- 什么是主键？
- 什么是索引？
- `LIMIT` 和 `OFFSET` 分别做什么？
- 为什么下单需要事务？
- 为什么 GORM 学习不能脱离 SQL？
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
