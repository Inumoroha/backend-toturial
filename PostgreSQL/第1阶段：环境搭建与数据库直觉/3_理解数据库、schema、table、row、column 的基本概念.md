# 3. 理解 database、schema、table、row、column 的基本概念

本节目标：理解 PostgreSQL 中 database、schema、table、column、row 这几个基础概念，并能通过简单 SQL 查看当前数据库、当前 schema，以及创建一个用于练习的 schema 和 table。

这些概念是后续学习建表、查询、约束、索引和 Go 项目接入 PostgreSQL 的基础。刚开始可以借助 Excel 类比理解，但要记住：数据库不只是“高级 Excel”，它更强调数据类型、约束、关系、查询能力和事务一致性。

---

## 一、整体结构

在 PostgreSQL 中，可以先用下面这句话建立整体直觉：

```text
一个 PostgreSQL 服务中可以有多个 database。
一个 database 中可以有多个 schema。
一个 schema 中可以有多张 table。
一张 table 由 column 定义结构，由 row 存储具体数据。
```

也可以理解为：

```text
PostgreSQL 服务
└── database
    └── schema
        └── table
            ├── column
            └── row
```

在本学习路线中，我们推荐使用第 1、2 节创建的 `learn_pg` 数据库作为练习环境。

---

## 二、Database：数据库

Database 是 PostgreSQL 中较大的数据隔离单位。通常一个独立项目会使用一个独立数据库。

例如：

- 博客系统可以使用 `blog` 数据库。
- 电商系统可以使用 `shop` 数据库。
- 当前学习环境可以使用 `learn_pg` 数据库。

查看当前连接的数据库：

```sql
SELECT current_database();
```

查看所有数据库，可以在 `psql` 中执行：

```text
\l
```

创建数据库：

```sql
CREATE DATABASE learn_pg;
```

注意：如果 `learn_pg` 已经存在，再次创建会报错。学习阶段看到这个错误不用紧张，说明数据库已经创建过了。

---

## 三、Schema：模式 / 命名空间

Schema 是 database 内部的命名空间，用来组织表、视图、函数等数据库对象，也可以避免命名冲突。

PostgreSQL 默认会提供一个名为 `public` 的 schema。如果你创建表时不指定 schema，表通常会被创建到 `public` 中。

例如下面两种写法通常等价：

```sql
CREATE TABLE users (
    id integer
);
```

```sql
CREATE TABLE public.users (
    id integer
);
```

查看当前 schema：

```sql
SELECT current_schema();
```

查看当前搜索路径：

```sql
SHOW search_path;
```

创建一个新的 schema：

```sql
CREATE SCHEMA demo;
```

在指定 schema 中创建表：

```sql
CREATE TABLE demo.users (
    id integer,
    name text
);
```

完整表名通常可以写成：

```text
schema_name.table_name
```

例如：

```text
demo.users
public.users
```

---

## 四、Table：数据表

Table 是真正存放同一类数据的地方。每张表应该表达一种相对明确的业务对象。

例如：

- `users` 表存用户。
- `posts` 表存文章。
- `orders` 表存订单。
- `products` 表存商品。

创建一张简单的用户表：

```sql
CREATE TABLE demo.users (
    id integer,
    name text,
    email text
);
```

查看当前数据库中的表，可以在 `psql` 中执行：

```text
\dt
```

查看指定 schema 中的表：

```text
\dt demo.*
```

查看表结构：

```text
\d demo.users
```

注意：真实项目中的表通常还会包含主键、唯一约束、非空约束、时间字段等内容。这里先用最简单的结构帮助理解概念。

---

## 五、Column：列 / 字段

Column 定义一张表可以存哪些属性，也规定每个属性的数据类型。

例如在 `demo.users` 表中：

```sql
CREATE TABLE demo.users (
    id integer,
    name text,
    email text
);
```

这里有三列：

- `id`：用户 ID，类型是 `integer`。
- `name`：用户名，类型是 `text`。
- `email`：邮箱，类型是 `text`。

数据库中的列比 Excel 更严格。每一列都有明确的数据类型，数据库会拒绝不符合类型要求的数据。

例如 `id` 是 `integer`，它应该存整数，不应该存普通字符串。

---

## 六、Row：行 / 记录

Row 是表中的一条具体数据，也常被称为一条记录。

向 `demo.users` 插入一行数据：

```sql
INSERT INTO demo.users (id, name, email)
VALUES (1, '张三', 'zhangsan@example.com');
```

再插入一行：

```sql
INSERT INTO demo.users (id, name, email)
VALUES (2, '李四', 'lisi@example.com');
```

查询表中的所有行：

```sql
SELECT * FROM demo.users;
```

查询结果中，每一行都代表一个用户，每一列代表这个用户的一个属性。

---

## 七、完整练习示例

下面是一组可以直接执行的 SQL，用来串联本节概念。

创建 schema：

```sql
CREATE SCHEMA demo;
```

创建 table：

```sql
CREATE TABLE demo.users (
    id integer,
    name text,
    email text
);
```

插入 row：

```sql
INSERT INTO demo.users (id, name, email)
VALUES
    (1, '张三', 'zhangsan@example.com'),
    (2, '李四', 'lisi@example.com');
```

查询数据：

```sql
SELECT id, name, email
FROM demo.users;
```

查看当前 database 和 schema：

```sql
SELECT current_database();
SELECT current_schema();
SHOW search_path;
```

如果你重复执行创建 schema 或创建 table 的语句，可能会看到“已经存在”的错误。后续学习中会介绍如何使用 `IF NOT EXISTS` 处理这类情况。

---

## 八、常见误区

### 1. database 和 schema 是一回事吗？

不是。

Database 是更大的隔离单位。Schema 是 database 内部的命名空间。

通常可以理解为：

```text
database > schema > table
```

### 2. PostgreSQL 的 schema 和 MySQL 的 database 一样吗？

不完全一样。

在 MySQL 中，很多时候 database 和 schema 的概念接近；但在 PostgreSQL 中，database 和 schema 是更明确的两层结构。

学习 PostgreSQL 时，建议按 PostgreSQL 的模型理解：

```text
一个 database 里面可以有多个 schema。
```

### 3. 不写 schema 会怎样？

如果创建表时不写 schema，PostgreSQL 通常会根据 `search_path` 把表创建到默认 schema 中，常见情况是 `public`。

例如：

```sql
CREATE TABLE users (
    id integer
);
```

通常会创建为：

```text
public.users
```

### 4. table 和 row 是什么关系？

Table 定义数据结构，row 是表中的具体数据。

例如 `users` 是表，`张三` 这一条用户信息就是表中的一行。

### 5. column 和 row 谁决定数据结构？

Column 决定结构，row 存储具体数据。

例如 `id`、`name`、`email` 是列；`1, 张三, zhangsan@example.com` 是一行数据。

---

## 九、本节达标标准

学完本节后，你应该能够做到：

- 解释 database、schema、table、column、row 分别是什么。
- 说清楚 `database > schema > table` 的层级关系。
- 知道 PostgreSQL 默认 schema 通常是 `public`。
- 知道完整表名可以写成 `schema_name.table_name`。
- 能执行 `SELECT current_database();` 查看当前数据库。
- 能执行 `SELECT current_schema();` 查看当前 schema。
- 能创建一个简单 schema 和 table。
- 能向表中插入 row，并用 `SELECT` 查询出来。

掌握这些概念之后，就可以进入下一节：理解 PostgreSQL 与 MySQL、SQLite、MongoDB 的区别。