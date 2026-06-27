# 2. DML：插入、修改和删除数据

本节目标：理解 DML 是什么，掌握 PostgreSQL 中常见的 `INSERT`、`UPDATE`、`DELETE` 语句，并能在 `demo.users` 表中完成新增、修改和删除数据。

上一节 DDL 负责定义表结构，本节 DML 负责操作表中的具体数据。后端开发中，创建用户、更新资料、删除记录，本质上都离不开 DML。

---

## 一、DML 是什么

DML 是 Data Manipulation Language 的缩写，中文通常叫“数据操作语言”。

它主要负责操作表中的行数据，例如：

- 插入数据：`INSERT`
- 修改数据：`UPDATE`
- 删除数据：`DELETE`

可以简单理解为：

```text
DDL 负责定义结构。
DML 负责操作数据。
DQL 负责查询数据。
```

本节先学习 `INSERT`、`UPDATE`、`DELETE`。查询数据的 `SELECT` 会在后续单独学习。

---

## 二、准备练习表

请先连接到学习数据库 `learn_pg`。

查看当前数据库：

```sql
SELECT current_database();
```

创建练习用 schema：

```sql
CREATE SCHEMA IF NOT EXISTS demo;
```

创建练习用表：

```sql
DROP TABLE IF EXISTS demo.users;

CREATE TABLE demo.users (
    id integer,
    username text,
    email text,
    age integer,
    created_at timestamptz DEFAULT now()
);
```

查看表结构：

```text
\d demo.users
```

注意：这张表暂时没有主键和唯一约束，是为了让你先专注于 DML 语法。后续会专门学习主键、唯一约束和表设计。

---

## 三、INSERT：插入数据

`INSERT` 用来向表中插入新的行。

### 1. 插入一行数据

```sql
INSERT INTO demo.users (id, username, email, age)
VALUES (1, 'zhangsan', 'zhangsan@example.com', 18);
```

查询数据：

```sql
SELECT * FROM demo.users;
```

### 2. 插入多行数据

```sql
INSERT INTO demo.users (id, username, email, age)
VALUES
    (2, 'lisi', 'lisi@example.com', 20),
    (3, 'wangwu', 'wangwu@example.com', 22),
    (4, 'zhaoliu', 'zhaoliu@example.com', 25);
```

查询数据：

```sql
SELECT * FROM demo.users;
```

### 3. 省略有默认值的列

`created_at` 设置了默认值 `now()`，插入时可以不手动传入：

```sql
INSERT INTO demo.users (id, username, email, age)
VALUES (5, 'sunqi', 'sunqi@example.com', 28);
```

查看结果：

```sql
SELECT id, username, email, age, created_at
FROM demo.users;
```

### 4. 使用 RETURNING 返回插入结果

PostgreSQL 支持 `RETURNING`，可以在插入后直接返回插入的数据：

```sql
INSERT INTO demo.users (id, username, email, age)
VALUES (6, 'zhouba', 'zhouba@example.com', 30)
RETURNING id, username, created_at;
```

这在后端开发中很常见。例如创建用户后，立即拿到新用户 ID 和创建时间。

---

## 四、UPDATE：修改数据

`UPDATE` 用来修改表中已有的行。

### 1. 修改单个用户

把 `id = 1` 的用户年龄改成 19：

```sql
UPDATE demo.users
SET age = 19
WHERE id = 1;
```

查看结果：

```sql
SELECT * FROM demo.users
WHERE id = 1;
```

### 2. 同时修改多个列

修改用户名和邮箱：

```sql
UPDATE demo.users
SET username = 'zhangsan_new',
    email = 'zhangsan_new@example.com'
WHERE id = 1;
```

### 3. 使用表达式更新

让所有年龄大于等于 20 的用户年龄加 1：

```sql
UPDATE demo.users
SET age = age + 1
WHERE age >= 20;
```

查看结果：

```sql
SELECT id, username, age
FROM demo.users
ORDER BY id;
```

### 4. 使用 RETURNING 查看更新结果

```sql
UPDATE demo.users
SET age = age + 1
WHERE id = 2
RETURNING id, username, age;
```

`RETURNING` 可以帮助你确认到底更新了哪些行。

### 5. UPDATE 一定要小心 WHERE

下面这条语句非常危险：

```sql
UPDATE demo.users
SET age = 18;
```

因为它没有 `WHERE`，会把整张表所有用户的年龄都改成 18。

学习阶段可以理解它的效果，但真实项目中执行 `UPDATE` 前一定要先确认条件。

---

## 五、DELETE：删除数据

`DELETE` 用来删除表中的行。

### 1. 删除指定用户

删除 `id = 6` 的用户：

```sql
DELETE FROM demo.users
WHERE id = 6;
```

查看结果：

```sql
SELECT * FROM demo.users
ORDER BY id;
```

### 2. 使用 RETURNING 查看删除结果

```sql
DELETE FROM demo.users
WHERE id = 5
RETURNING id, username, email;
```

这样可以看到刚刚被删除的是哪条数据。

### 3. DELETE 一定要小心 WHERE

下面这条语句非常危险：

```sql
DELETE FROM demo.users;
```

它会删除 `demo.users` 表中的所有行，但表结构还在。

如果你只想删除某个用户，必须写清楚条件：

```sql
DELETE FROM demo.users
WHERE id = 3;
```

---

## 六、DELETE、DROP、TRUNCATE 的区别

这三个命令都可能让数据消失，但含义不同。

| 命令 | 类型 | 作用 | 是否保留表结构 | 常见用途 |
| --- | --- | --- | --- | --- |
| `DELETE` | DML | 删除表中的行 | 保留 | 按条件删除数据 |
| `DROP TABLE` | DDL | 删除整张表 | 不保留 | 删除不再需要的表 |
| `TRUNCATE` | 通常归为 DDL 或特殊命令 | 快速清空整张表 | 保留 | 清空测试数据 |

示例：

```sql
DELETE FROM demo.users WHERE id = 1;
```

```sql
DROP TABLE demo.users;
```

```sql
TRUNCATE TABLE demo.users;
```

初学阶段建议优先使用 `DELETE ... WHERE ...`，不要随便执行 `DROP TABLE` 或 `TRUNCATE`。

---

## 七、DML 与事务

DML 经常需要配合事务使用。

例如你想尝试删除数据，但不确定结果，可以先开启事务：

```sql
BEGIN;

DELETE FROM demo.users
WHERE age < 21;

SELECT * FROM demo.users
ORDER BY id;

ROLLBACK;
```

执行 `ROLLBACK` 后，刚才的删除会撤销。

如果确认修改没问题，可以使用：

```sql
BEGIN;

UPDATE demo.users
SET age = age + 1
WHERE id = 2;

COMMIT;
```

后续学习事务时会更系统地讲 `BEGIN`、`COMMIT`、`ROLLBACK`。

---

## 八、完整练习

可以从头执行下面这组 SQL。

重建练习表：

```sql
DROP TABLE IF EXISTS demo.users;

CREATE TABLE demo.users (
    id integer,
    username text,
    email text,
    age integer,
    created_at timestamptz DEFAULT now()
);
```

插入数据：

```sql
INSERT INTO demo.users (id, username, email, age)
VALUES
    (1, 'zhangsan', 'zhangsan@example.com', 18),
    (2, 'lisi', 'lisi@example.com', 20),
    (3, 'wangwu', 'wangwu@example.com', 22);
```

修改数据：

```sql
UPDATE demo.users
SET age = 21
WHERE id = 2
RETURNING id, username, age;
```

删除数据：

```sql
DELETE FROM demo.users
WHERE id = 3
RETURNING id, username;
```

查看最终结果：

```sql
SELECT id, username, email, age, created_at
FROM demo.users
ORDER BY id;
```

---

## 九、常见误区

### 1. INSERT 时可以不写列名吗？

可以，但不推荐。

不推荐写法：

```sql
INSERT INTO demo.users
VALUES (1, 'zhangsan', 'zhangsan@example.com', 18, now());
```

推荐写法：

```sql
INSERT INTO demo.users (id, username, email, age)
VALUES (1, 'zhangsan', 'zhangsan@example.com', 18);
```

写清楚列名更安全，也更容易维护。

### 2. UPDATE 忘记 WHERE 会怎样？

会更新整张表。

```sql
UPDATE demo.users
SET age = 18;
```

真实项目中这是非常危险的操作。

### 3. DELETE 忘记 WHERE 会怎样？

会删除整张表中的所有行。

```sql
DELETE FROM demo.users;
```

表结构还在，但数据会被清空。

### 4. DELETE 和 DROP 有什么区别？

`DELETE` 删除行，表还在。

`DROP TABLE` 删除整张表，表结构和数据都没了。

### 5. 为什么 PostgreSQL 的 RETURNING 很有用？

`RETURNING` 可以让你在插入、更新、删除后，立即拿到受影响的数据。

后端开发中常用于：

- 创建用户后返回用户 ID。
- 更新资料后返回最新数据。
- 删除数据前确认被删的是哪一行。

---

## 十、本节达标标准

学完本节后，你应该能够做到：

- 解释 DML 是什么。
- 区分 DDL、DML、DQL。
- 使用 `INSERT` 插入一行或多行数据。
- 使用 `UPDATE ... WHERE ...` 修改指定数据。
- 使用 `DELETE ... WHERE ...` 删除指定数据。
- 使用 `RETURNING` 查看插入、更新或删除的结果。
- 说清楚 `DELETE`、`DROP TABLE`、`TRUNCATE` 的区别。
- 知道 `UPDATE` 和 `DELETE` 忘记 `WHERE` 的风险。
- 能用事务测试修改，并通过 `ROLLBACK` 撤销。

掌握 DML 后，就可以进入下一节：DQL，也就是使用 `SELECT` 查询数据。