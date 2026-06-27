# 1. DDL：定义数据库结构

本节目标：理解 DDL 是什么，掌握 PostgreSQL 中常见的 `CREATE`、`ALTER`、`DROP` 语句，并能创建、修改和删除用于练习的 schema 与 table。

DDL 是 SQL 中负责定义数据库结构的一类语句。后续学习 DML、查询、约束、索引之前，必须先理解如何创建和调整表结构。

---

## 一、DDL 是什么

DDL 是 Data Definition Language 的缩写，中文通常叫“数据定义语言”。

它主要负责定义或修改数据库对象的结构，例如：

- database
- schema
- table
- column
- view
- index
- constraint

可以简单理解为：

```text
DDL 负责定义结构。
DML 负责操作数据。
DQL 负责查询数据。
```

例如：

- 创建一张用户表，是 DDL。
- 向用户表插入一条用户数据，是 DML。
- 查询用户列表，是 DQL。

---

## 二、DDL 和 DML 的区别

| 类型 | 全称 | 作用 | 常见命令 |
| --- | --- | --- | --- |
| DDL | Data Definition Language | 定义结构 | `CREATE`、`ALTER`、`DROP` |
| DML | Data Manipulation Language | 操作数据 | `INSERT`、`UPDATE`、`DELETE` |
| DQL | Data Query Language | 查询数据 | `SELECT` |

本节只学习 DDL。下一节再学习如何向表中插入、修改、删除和查询数据。

---

## 三、准备练习环境

请先确认你已经连接到学习数据库 `learn_pg`。

查看当前数据库：

```sql
SELECT current_database();
```

建议本节继续使用 `demo` schema。先创建它：

```sql
CREATE SCHEMA IF NOT EXISTS demo;
```

`IF NOT EXISTS` 的作用是：如果 `demo` 已经存在，就不重复创建，也不会报错。

---

## 四、CREATE：创建数据库对象

`CREATE` 用来创建新的数据库对象。

常见用途包括：

- 创建 database
- 创建 schema
- 创建 table
- 创建 index
- 创建 view

本阶段重点先掌握 schema 和 table。

### 1. 创建 schema

```sql
CREATE SCHEMA IF NOT EXISTS demo;
```

查看当前 schema：

```sql
SELECT current_schema();
```

查看搜索路径：

```sql
SHOW search_path;
```

### 2. 创建 table

创建一张简单的用户表：

```sql
CREATE TABLE IF NOT EXISTS demo.users (
    id integer,
    name text,
    email text,
    created_at timestamptz
);
```

这条 SQL 的含义：

- `demo.users`：在 `demo` schema 下创建 `users` 表。
- `id integer`：创建 `id` 列，类型是整数。
- `name text`：创建 `name` 列，类型是文本。
- `email text`：创建 `email` 列，类型是文本。
- `created_at timestamptz`：创建 `created_at` 列，类型是带时区的时间。

在 `psql` 中查看表：

```text
\dt demo.*
```

查看表结构：

```text
\d demo.users
```

注意：这张表还没有主键、唯一约束和非空约束。这里先用最简单的结构学习 DDL，后续会专门学习约束和表设计。

---

## 五、ALTER：修改已有结构

`ALTER` 用来修改已经存在的数据库对象。

常见用途包括：

- 给表增加列。
- 删除列。
- 修改列类型。
- 重命名表。
- 重命名列。
- 增加或删除约束。

### 1. 给表增加列

给 `demo.users` 增加 `age` 列：

```sql
ALTER TABLE demo.users
ADD COLUMN age integer;
```

查看表结构：

```text
\d demo.users
```

### 2. 重命名列

把 `name` 列改名为 `username`：

```sql
ALTER TABLE demo.users
RENAME COLUMN name TO username;
```

### 3. 修改列类型

把 `email` 的类型从 `text` 改成 `varchar(255)`：

```sql
ALTER TABLE demo.users
ALTER COLUMN email TYPE varchar(255);
```

注意：修改列类型时，如果表中已经有数据，并且数据无法转换成新类型，就会失败。

### 4. 给列设置默认值

给 `created_at` 设置默认值：

```sql
ALTER TABLE demo.users
ALTER COLUMN created_at SET DEFAULT now();
```

以后插入数据时，如果没有指定 `created_at`，PostgreSQL 会默认使用当前时间。

### 5. 删除列

删除 `age` 列：

```sql
ALTER TABLE demo.users
DROP COLUMN age;
```

注意：删除列会删除该列中的所有数据。生产环境中执行前必须谨慎确认。

### 6. 重命名表

把 `demo.users` 改名为 `demo.app_users`：

```sql
ALTER TABLE demo.users
RENAME TO app_users;
```

再改回 `demo.users`：

```sql
ALTER TABLE demo.app_users
RENAME TO users;
```

---

## 六、DROP：删除数据库对象

`DROP` 用来删除数据库对象。

常见用途包括：

- 删除 table。
- 删除 schema。
- 删除 database。
- 删除 index。
- 删除 view。

### 1. 删除表

删除练习表：

```sql
DROP TABLE IF EXISTS demo.users;
```

`IF EXISTS` 的作用是：如果表不存在，也不会报错。

注意：`DROP TABLE` 会删除表结构，也会删除表里的全部数据。

### 2. 删除 schema

如果 schema 为空，可以删除：

```sql
DROP SCHEMA IF EXISTS demo;
```

如果 schema 中还有表，直接删除会失败。可以使用：

```sql
DROP SCHEMA IF EXISTS demo CASCADE;
```

注意：`CASCADE` 会连同 schema 下面的对象一起删除，包括表和数据。学习环境可以尝试，生产环境必须非常谨慎。

---

## 七、PostgreSQL 中 DDL 与事务

在 PostgreSQL 中，大多数 DDL 可以放在事务中执行，并且可以回滚。

示例：

```sql
BEGIN;

CREATE TABLE demo.temp_table (
    id integer
);

ROLLBACK;
```

执行 `ROLLBACK` 后，`demo.temp_table` 不会被保留。

你可以用下面命令验证：

```text
\dt demo.*
```

这和某些数据库不同。在一些数据库中，DDL 可能会隐式提交，执行后不容易回滚。

注意：PostgreSQL 并不是所有 DDL 都能放在事务块里执行。例如某些数据库级操作或并发索引操作会有额外限制。当前阶段只需要记住：PostgreSQL 的大多数表结构变更可以在事务中回滚。

---

## 八、完整练习

下面是一组完整练习，可以从头执行一遍。

创建 schema：

```sql
CREATE SCHEMA IF NOT EXISTS demo;
```

创建表：

```sql
CREATE TABLE IF NOT EXISTS demo.users (
    id integer,
    name text,
    email text,
    created_at timestamptz DEFAULT now()
);
```

增加列：

```sql
ALTER TABLE demo.users
ADD COLUMN age integer;
```

重命名列：

```sql
ALTER TABLE demo.users
RENAME COLUMN name TO username;
```

查看表结构：

```text
\d demo.users
```

删除练习表：

```sql
DROP TABLE IF EXISTS demo.users;
```

如果你想保留这张表给下一节 DML 使用，可以暂时不要执行最后的 `DROP TABLE`。

---

## 九、常见误区

### 1. DDL 会操作具体数据吗？

DDL 主要操作结构，不负责插入、修改、删除具体行数据。

例如：

- `CREATE TABLE` 是 DDL。
- `INSERT INTO` 是 DML。

### 2. DROP 和 DELETE 是一回事吗？

不是。

- `DROP TABLE` 删除整张表，包括结构和数据。
- `DELETE FROM table` 删除表里的行，但表结构还在。

### 3. ALTER TABLE 一定安全吗？

不一定。

`ALTER TABLE` 会修改表结构。某些操作可能影响已有数据，甚至在大表上造成锁等待。学习阶段可以大胆练习，真实项目中必须谨慎执行。

### 4. 为什么建议写 IF EXISTS 或 IF NOT EXISTS？

它们可以让练习 SQL 更容易重复执行。

例如：

```sql
CREATE SCHEMA IF NOT EXISTS demo;
DROP TABLE IF EXISTS demo.users;
```

这样重复执行时不容易因为对象已经存在或不存在而中断。

### 5. 可以直接 DROP DATABASE 吗？

学习阶段不建议随便执行 `DROP DATABASE`。

删除数据库会删除整个数据库中的对象和数据。除非你明确知道自己在删除什么，否则不要执行。

---

## 十、本节达标标准

学完本节后，你应该能够做到：

- 解释 DDL 是什么。
- 区分 DDL、DML、DQL。
- 使用 `CREATE SCHEMA` 创建 schema。
- 使用 `CREATE TABLE` 创建表。
- 使用 `ALTER TABLE` 增加列、重命名列、修改列类型。
- 使用 `DROP TABLE` 删除练习表。
- 知道 `DROP` 和 `DELETE` 的区别。
- 知道 PostgreSQL 的大多数 DDL 可以在事务中回滚。
- 知道结构变更和删除操作在生产环境中需要谨慎。

掌握 DDL 后，就可以进入下一节：DML，学习如何向表中插入、修改和删除数据。