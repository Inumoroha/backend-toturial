# 3. DQL：使用 SELECT 查询数据

本节目标：理解 DQL 是什么，掌握 PostgreSQL 中最常用的 `SELECT` 查询语句，能够完成基础查询、条件查询、排序、分页、去重和简单表达式查询。

前两节中，DDL 负责定义表结构，DML 负责插入、修改、删除数据。本节开始学习 DQL，也就是如何从表中查出你想要的数据。

---

## 一、DQL 是什么

DQL 是 Data Query Language 的缩写，中文通常叫“数据查询语言”。

在日常开发中，DQL 主要就是：

```sql
SELECT
```

可以简单理解为：

```text
DDL 负责定义结构。
DML 负责操作数据。
DQL 负责查询数据。
```

后端开发里，绝大多数接口都离不开查询。例如：

- 查询用户信息。
- 查询文章列表。
- 查询订单详情。
- 查询分页数据。
- 查询符合条件的数据。

---

## 二、准备练习数据

请先连接到学习数据库 `learn_pg`。

创建 schema：

```sql
CREATE SCHEMA IF NOT EXISTS demo;
```

重建练习表：

```sql
DROP TABLE IF EXISTS demo.users;

CREATE TABLE demo.users (
    id integer,
    username text,
    email text,
    age integer,
    city text,
    is_active boolean,
    created_at timestamptz DEFAULT now()
);
```

插入练习数据：

```sql
INSERT INTO demo.users (id, username, email, age, city, is_active, created_at)
VALUES
    (1, 'zhangsan', 'zhangsan@example.com', 18, 'Beijing', true,  '2026-01-01 10:00:00+08'),
    (2, 'lisi',     'lisi@example.com',     20, 'Shanghai', true,  '2026-01-02 10:00:00+08'),
    (3, 'wangwu',   'wangwu@example.com',   22, 'Beijing', false, '2026-01-03 10:00:00+08'),
    (4, 'zhaoliu',  'zhaoliu@example.com',  25, 'Shenzhen', true, '2026-01-04 10:00:00+08'),
    (5, 'sunqi',    'sunqi@example.com',    28, 'Shanghai', false,'2026-01-05 10:00:00+08'),
    (6, 'zhouba',   'zhouba@example.com',   30, 'Beijing', true,  '2026-01-06 10:00:00+08');
```

查看表结构：

```text
\d demo.users
```

---

## 三、最基础的 SELECT

查询整张表：

```sql
SELECT *
FROM demo.users;
```

`*` 表示查询所有列。

实际开发中，不建议长期使用 `SELECT *`。更推荐明确写出需要的列：

```sql
SELECT id, username, email
FROM demo.users;
```

这样有几个好处：

- 查询结果更清楚。
- 避免返回不需要的字段。
- 表结构变化时影响更小。
- 后端接口字段更容易控制。

---

## 四、给列起别名

可以使用 `AS` 给查询结果中的列起别名：

```sql
SELECT
    id AS user_id,
    username AS name,
    email
FROM demo.users;
```

`AS` 可以省略，但初学阶段建议写上，更清晰：

```sql
SELECT
    id user_id,
    username name
FROM demo.users;
```

上面这种写法能运行，但不如显式写 `AS` 易读。

---

## 五、WHERE：条件查询

`WHERE` 用来筛选符合条件的行。

### 1. 等值查询

查询城市是 Beijing 的用户：

```sql
SELECT id, username, city
FROM demo.users
WHERE city = 'Beijing';
```

### 2. 比较查询

查询年龄大于 22 的用户：

```sql
SELECT id, username, age
FROM demo.users
WHERE age > 22;
```

常见比较操作符：

```text
=     等于
<>    不等于
>     大于
>=    大于等于
<     小于
<=    小于等于
```

### 3. 布尔值查询

查询启用状态的用户：

```sql
SELECT id, username, is_active
FROM demo.users
WHERE is_active = true;
```

也可以写成：

```sql
SELECT id, username, is_active
FROM demo.users
WHERE is_active;
```

查询未启用的用户：

```sql
SELECT id, username, is_active
FROM demo.users
WHERE is_active = false;
```

---

## 六、AND、OR、NOT：组合条件

### 1. AND

查询城市是 Beijing，并且年龄大于 20 的用户：

```sql
SELECT id, username, age, city
FROM demo.users
WHERE city = 'Beijing'
  AND age > 20;
```

### 2. OR

查询城市是 Beijing 或 Shanghai 的用户：

```sql
SELECT id, username, city
FROM demo.users
WHERE city = 'Beijing'
   OR city = 'Shanghai';
```

### 3. NOT

查询不是 Beijing 的用户：

```sql
SELECT id, username, city
FROM demo.users
WHERE NOT city = 'Beijing';
```

更常见写法是：

```sql
SELECT id, username, city
FROM demo.users
WHERE city <> 'Beijing';
```

### 4. 使用括号控制优先级

查询启用状态，并且城市是 Beijing 或 Shanghai 的用户：

```sql
SELECT id, username, city, is_active
FROM demo.users
WHERE is_active = true
  AND (city = 'Beijing' OR city = 'Shanghai');
```

有 `AND` 和 `OR` 混用时，建议主动加括号，避免理解错误。

---

## 七、IN、BETWEEN、LIKE

### 1. IN

查询城市在多个候选值中的用户：

```sql
SELECT id, username, city
FROM demo.users
WHERE city IN ('Beijing', 'Shanghai');
```

这比多个 `OR` 更简洁。

### 2. BETWEEN

查询年龄在 20 到 28 之间的用户：

```sql
SELECT id, username, age
FROM demo.users
WHERE age BETWEEN 20 AND 28;
```

注意：`BETWEEN 20 AND 28` 包含 20 和 28。

### 3. LIKE

查询邮箱中包含 `example` 的用户：

```sql
SELECT id, username, email
FROM demo.users
WHERE email LIKE '%example%';
```

查询用户名以 `zh` 开头的用户：

```sql
SELECT id, username
FROM demo.users
WHERE username LIKE 'zh%';
```

`LIKE` 中常用通配符：

```text
%    匹配任意长度字符
_    匹配单个字符
```

PostgreSQL 中如果想做不区分大小写的匹配，可以使用 `ILIKE`：

```sql
SELECT id, username, email
FROM demo.users
WHERE email ILIKE '%EXAMPLE%';
```

---

## 八、ORDER BY：排序

`ORDER BY` 用来对查询结果排序。

按年龄从小到大排序：

```sql
SELECT id, username, age
FROM demo.users
ORDER BY age ASC;
```

`ASC` 表示升序，可以省略。

按年龄从大到小排序：

```sql
SELECT id, username, age
FROM demo.users
ORDER BY age DESC;
```

按城市升序、年龄降序排序：

```sql
SELECT id, username, city, age
FROM demo.users
ORDER BY city ASC, age DESC;
```

---

## 九、LIMIT 和 OFFSET：限制数量与分页

`LIMIT` 用来限制返回多少行。

查询前 3 个用户：

```sql
SELECT id, username, age
FROM demo.users
ORDER BY id
LIMIT 3;
```

`OFFSET` 用来跳过前面多少行。

查询第 2 页，每页 3 条：

```sql
SELECT id, username, age
FROM demo.users
ORDER BY id
LIMIT 3 OFFSET 3;
```

分页公式：

```text
OFFSET = (page - 1) * page_size
```

例如第 3 页，每页 10 条：

```text
OFFSET = (3 - 1) * 10 = 20
```

注意：使用分页时，应该配合稳定的 `ORDER BY`，否则每次返回顺序可能不稳定。

---

## 十、DISTINCT：去重

查询所有出现过的城市：

```sql
SELECT DISTINCT city
FROM demo.users;
```

按城市排序：

```sql
SELECT DISTINCT city
FROM demo.users
ORDER BY city;
```

`DISTINCT` 会去除重复结果。它常用于查询某列有哪些不同取值。

---

## 十一、表达式查询

SELECT 不只能查询列，也可以查询表达式。

年龄加 1：

```sql
SELECT id, username, age, age + 1 AS next_age
FROM demo.users;
```

拼接文本：

```sql
SELECT username || ' <' || email || '>' AS display_name
FROM demo.users;
```

判断用户状态：

```sql
SELECT
    id,
    username,
    CASE
        WHEN is_active THEN 'active'
        ELSE 'inactive'
    END AS status
FROM demo.users;
```

`CASE WHEN` 在后端业务查询中很常见，可以把数据库中的值转换成更适合展示或处理的结果。

---

## 十二、NULL 的基础处理

后续会更系统讲 NULL，这里先掌握最基础的判断。

插入一条城市为空的数据：

```sql
INSERT INTO demo.users (id, username, email, age, city, is_active)
VALUES (7, 'nullcity', 'nullcity@example.com', 19, NULL, true);
```

查询城市为空的用户：

```sql
SELECT id, username, city
FROM demo.users
WHERE city IS NULL;
```

查询城市不为空的用户：

```sql
SELECT id, username, city
FROM demo.users
WHERE city IS NOT NULL;
```

注意：不要用下面这种写法判断 NULL：

```sql
WHERE city = NULL
```

这是错误理解。判断 NULL 要使用 `IS NULL` 或 `IS NOT NULL`。

---

## 十三、完整练习

查询所有用户：

```sql
SELECT id, username, email, age, city, is_active
FROM demo.users
ORDER BY id;
```

查询启用状态的 Beijing 用户：

```sql
SELECT id, username, city, is_active
FROM demo.users
WHERE city = 'Beijing'
  AND is_active = true
ORDER BY id;
```

查询年龄在 20 到 28 岁之间的用户：

```sql
SELECT id, username, age
FROM demo.users
WHERE age BETWEEN 20 AND 28
ORDER BY age;
```

查询不同城市：

```sql
SELECT DISTINCT city
FROM demo.users
WHERE city IS NOT NULL
ORDER BY city;
```

分页查询：

```sql
SELECT id, username, age
FROM demo.users
ORDER BY id
LIMIT 3 OFFSET 0;
```

---

## 十四、常见误区

### 1. 可以一直使用 SELECT * 吗？

学习阶段可以，真实项目中不建议长期使用。

更推荐明确写出需要的列：

```sql
SELECT id, username, email
FROM demo.users;
```

### 2. WHERE 可以写在 ORDER BY 后面吗？

不可以。

常见顺序是：

```sql
SELECT ...
FROM ...
WHERE ...
ORDER BY ...
LIMIT ...
OFFSET ...;
```

### 3. 判断 NULL 可以用 = NULL 吗？

不可以。

应该使用：

```sql
IS NULL
IS NOT NULL
```

### 4. LIMIT 不写 ORDER BY 可以吗？

语法上可以，但不推荐。

不写 `ORDER BY` 时，数据库不保证返回顺序稳定。分页查询尤其应该写明确排序。

### 5. LIKE '%xxx%' 有什么问题？

它能实现模糊搜索，但在数据量大时可能比较慢。后续学习索引和全文搜索时，会进一步讲如何优化搜索。

---

## 十五、本节达标标准

学完本节后，你应该能够做到：

- 解释 DQL 是什么。
- 使用 `SELECT` 查询指定列。
- 使用 `WHERE` 过滤数据。
- 使用 `AND`、`OR`、`NOT` 组合条件。
- 使用 `IN`、`BETWEEN`、`LIKE`、`ILIKE`。
- 使用 `ORDER BY` 排序。
- 使用 `LIMIT` 和 `OFFSET` 做简单分页。
- 使用 `DISTINCT` 去重。
- 使用 `IS NULL` 和 `IS NOT NULL` 判断空值。
- 写出一条包含查询列、条件、排序、分页的完整 SQL。

掌握基础 DQL 后，就可以继续学习 PostgreSQL 常见数据类型，理解整数、文本、布尔值、时间、UUID 等字段应该如何选择。