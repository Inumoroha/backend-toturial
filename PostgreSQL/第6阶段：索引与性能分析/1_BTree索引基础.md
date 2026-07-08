# 1. B-Tree 索引基础

本节目标：理解 B-Tree 索引的作用，掌握普通索引的创建、查看和删除，并知道 B-Tree 适合哪些查询。

B-Tree 是 PostgreSQL 默认、也是最常用的索引类型。

当你写：

```sql
CREATE INDEX users_email_idx
ON demo.perf_users (email);
```

如果没有特别指定索引类型，PostgreSQL 默认创建的就是 B-Tree 索引。

---

## 一、B-Tree 索引解决什么问题

没有索引时，查询某个邮箱：

```sql
SELECT id, username, email
FROM demo.perf_users
WHERE email = 'user_50000@example.com';
```

数据库可能需要扫描整张表：

```text
第 1 行，看 email 是否匹配。
第 2 行，看 email 是否匹配。
第 3 行，看 email 是否匹配。
...
```

如果表有 100000 行，最坏可能要检查很多行。

B-Tree 索引可以理解为一本排好序的目录：

```text
email -> 对应的数据位置
```

查找时，数据库可以在索引中快速定位目标值，而不是从头扫完整张表。

---

## 二、创建普通 B-Tree 索引

为用户邮箱创建索引：

```sql
CREATE INDEX perf_users_email_idx
ON demo.perf_users (email);
```

索引命名建议包含：

- 表名。
- 字段名。
- `_idx` 后缀。

例如：

```text
perf_users_email_idx
```

这样以后看执行计划或排查问题时，能快速知道这个索引用在哪张表、哪个字段上。

---

## 三、使用 EXPLAIN 观察索引效果

建索引前后可以用 `EXPLAIN` 对比：

```sql
EXPLAIN
SELECT id, username, email
FROM demo.perf_users
WHERE email = 'user_50000@example.com';
```

如果没有索引，可能看到：

```text
Seq Scan on perf_users
  Filter: (email = 'user_50000@example.com'::text)
```

建索引后，可能看到：

```text
Index Scan using perf_users_email_idx on perf_users
  Index Cond: (email = 'user_50000@example.com'::text)
```

先不用纠结每个数字。当前重点是看扫描方式：

- `Seq Scan`：顺序扫描表。
- `Index Scan`：使用索引扫描。

---

## 四、B-Tree 适合的查询

B-Tree 索引适合很多常见条件。

### 1. 等值查询

```sql
SELECT *
FROM demo.perf_users
WHERE email = 'user_50000@example.com';
```

等值查询是 B-Tree 最常见的使用场景。

### 2. 范围查询

```sql
SELECT *
FROM demo.perf_orders
WHERE created_at >= '2026-01-01'
  AND created_at < '2026-02-01';
```

可以创建：

```sql
CREATE INDEX perf_orders_created_at_idx
ON demo.perf_orders (created_at);
```

B-Tree 中的数据是有序的，所以很适合范围查询。

### 3. 排序

```sql
SELECT id, order_no, created_at
FROM demo.perf_orders
ORDER BY created_at DESC
LIMIT 20;
```

可以创建：

```sql
CREATE INDEX perf_orders_created_at_desc_idx
ON demo.perf_orders (created_at DESC);
```

如果索引顺序能满足 `ORDER BY`，PostgreSQL 可能不用额外排序。

### 4. 前缀 LIKE 查询

```sql
SELECT id, username
FROM demo.perf_users
WHERE username LIKE 'user_100%';
```

某些情况下，B-Tree 可以帮助前缀匹配。

但下面这种包含前导通配符的查询通常不能直接利用普通 B-Tree：

```sql
SELECT id, username
FROM demo.perf_users
WHERE username LIKE '%100%';
```

因为 `%100%` 不是从字符串开头定位，普通 B-Tree 很难快速跳到目标范围。

---

## 五、给外键列创建索引

外键约束本身不等于外键列一定有索引。

例如文章表：

```sql
SELECT id, user_id, title, created_at
FROM demo.perf_posts
WHERE user_id = 100
ORDER BY created_at DESC
LIMIT 20;
```

如果经常根据 `user_id` 查文章，应该考虑索引：

```sql
CREATE INDEX perf_posts_user_id_idx
ON demo.perf_posts (user_id);
```

但这个查询还有排序：

```sql
ORDER BY created_at DESC
```

后面学习组合索引时，会进一步优化成：

```sql
CREATE INDEX perf_posts_user_id_created_at_idx
ON demo.perf_posts (user_id, created_at DESC);
```

当前先记住：

```text
外键保证引用关系。
索引提升查询性能。
二者不是一回事。
```

---

## 六、查看表上的索引

在 `psql` 中可以使用：

```sql
\d demo.perf_users
```

也可以查询系统视图：

```sql
SELECT
    schemaname,
    tablename,
    indexname,
    indexdef
FROM pg_indexes
WHERE schemaname = 'demo'
  AND tablename = 'perf_users'
ORDER BY indexname;
```

输出中的 `indexdef` 会显示索引定义。

---

## 七、删除索引

删除索引：

```sql
DROP INDEX demo.perf_users_email_idx;
```

如果不确定索引是否存在：

```sql
DROP INDEX IF EXISTS demo.perf_users_email_idx;
```

注意：删除索引会影响依赖它的查询性能。生产环境删除索引前要确认它确实无用。

---

## 八、索引会影响写入

给 `email` 建索引后，每次插入用户：

```sql
INSERT INTO demo.perf_users (username, email)
VALUES ('new_user', 'new_user@example.com');
```

PostgreSQL 不仅要插入表数据，还要维护 `perf_users_email_idx`。

更新索引列也有成本：

```sql
UPDATE demo.perf_users
SET email = 'new_email@example.com'
WHERE id = 1;
```

所以索引不是免费午餐。

原则是：

```text
读多写少、查询条件稳定、选择性好的字段，更适合建索引。
写入非常频繁且很少用于查询的字段，不要随便建索引。
```

---

## 九、什么是选择性

选择性可以粗略理解为：

```text
一个条件能过滤掉多少数据。
```

例如：

```sql
WHERE email = 'user_50000@example.com'
```

通常只返回 1 行，选择性很好。

而：

```sql
WHERE status = 'active'
```

如果 95% 用户都是 active，选择性就很差。

选择性差的条件，即使有索引，PostgreSQL 也可能选择顺序扫描，因为走索引再回表取大量数据可能更慢。

---

## 十、常见误区

### 1. 每个字段都应该建索引吗？

不是。

索引有写入成本和存储成本。应该为真实高频查询建索引，而不是给所有字段都建。

### 2. 建了索引就一定走索引吗？

不一定。

PostgreSQL 优化器会根据统计信息估算成本。如果它认为顺序扫描更便宜，就不会使用索引。

### 3. B-Tree 能解决所有搜索问题吗？

不能。

例如全文搜索、任意位置模糊搜索、JSONB 包含查询等，可能需要其他索引类型。当前阶段先掌握 B-Tree，因为它覆盖了大量后端常见查询。

### 4. 主键需要自己再建索引吗？

通常不需要。

PostgreSQL 会自动为 `PRIMARY KEY` 创建唯一 B-Tree 索引。

---

## 十一、本节达标标准

学完本节后，你应该能够做到：

- 解释 B-Tree 索引像一份有序目录。
- 使用 `CREATE INDEX` 创建普通索引。
- 使用 `EXPLAIN` 观察查询是否使用索引。
- 知道 B-Tree 适合等值查询、范围查询、排序和部分前缀匹配。
- 知道外键和索引不是一回事。
- 使用 `pg_indexes` 查看索引定义。
- 使用 `DROP INDEX` 删除索引。
- 知道索引会增加写入成本。
- 理解选择性对索引使用的影响。
