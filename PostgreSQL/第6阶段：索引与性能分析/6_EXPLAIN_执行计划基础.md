# 6. EXPLAIN：执行计划基础

本节目标：掌握 `EXPLAIN` 的基本用法，能看懂执行计划中的扫描方式、成本、估算行数和过滤条件。

`EXPLAIN` 用来查看 PostgreSQL 打算如何执行一条 SQL。

它不会真正执行普通查询，只展示计划。

```sql
EXPLAIN
SELECT id, username, email
FROM demo.perf_users
WHERE email = 'user_50000@example.com';
```

你可以把 `EXPLAIN` 理解为：

```text
让 PostgreSQL 先说出它准备怎么查。
```

---

## 一、为什么要看 EXPLAIN

遇到慢查询时，不能只靠猜：

```text
是不是没加索引？
是不是数据太多？
是不是排序慢？
是不是 JOIN 太慢？
```

`EXPLAIN` 能告诉你：

- 是否顺序扫描整张表。
- 是否使用索引。
- 是否需要排序。
- 是否使用嵌套循环、哈希连接等 JOIN 方式。
- 估算会返回多少行。
- PostgreSQL 认为各步骤成本有多高。

本阶段先重点关注：

- `Seq Scan`
- `Index Scan`
- `Bitmap Index Scan`
- `Sort`
- `Filter`
- `Index Cond`
- `cost`
- `rows`

---

## 二、基本语法

```sql
EXPLAIN
SELECT ...
```

例子：

```sql
EXPLAIN
SELECT id, username, email
FROM demo.perf_users
WHERE email = 'user_50000@example.com';
```

如果没有邮箱索引，可能看到：

```text
Seq Scan on perf_users  (cost=0.00..2185.00 rows=1 width=48)
  Filter: (email = 'user_50000@example.com'::text)
```

如果有邮箱索引，可能看到：

```text
Index Scan using perf_users_email_idx on perf_users  (cost=0.42..8.44 rows=1 width=48)
  Index Cond: (email = 'user_50000@example.com'::text)
```

不同机器、不同 PostgreSQL 版本、不同数据量，数字可能不一样。学习时先看结构，不要死记数字。

---

## 三、扫描方式

### 1. Seq Scan

```text
Seq Scan on perf_users
```

表示顺序扫描表。

也就是从表里读很多行，再逐行判断是否满足条件。

### 2. Index Scan

```text
Index Scan using perf_users_email_idx on perf_users
```

表示使用索引扫描。

通常会先查索引，再根据索引定位表中的行。

### 3. Bitmap Index Scan

```text
Bitmap Index Scan on perf_orders_status_idx
```

通常会配合：

```text
Bitmap Heap Scan
```

它会先通过索引找到一批可能的行位置，形成位图，再批量访问表。

后面会专门讲这几种扫描方式。

---

## 四、cost 是什么

执行计划里常见：

```text
cost=0.42..8.44
```

这两个数字表示估算成本：

```text
启动成本..总成本
```

- 启动成本：返回第一行前大概需要的成本。
- 总成本：返回全部结果大概需要的成本。

注意：

```text
cost 不是毫秒。
cost 是 PostgreSQL 内部用于比较计划优劣的估算单位。
```

优化器会比较不同执行方案的成本，然后选择它认为成本最低的方案。

---

## 五、rows 是什么

执行计划里：

```text
rows=1
```

表示 PostgreSQL 估算这一步会返回多少行。

例如：

```text
Seq Scan on perf_users  (cost=0.00..2185.00 rows=1 width=48)
```

这里 `rows=1` 表示优化器估算最终只有 1 行满足条件。

如果估算行数和真实行数差异很大，执行计划可能变差。

真实行数需要用 `EXPLAIN ANALYZE` 看。

---

## 六、width 是什么

```text
width=48
```

表示 PostgreSQL 估算每行平均宽度大约是多少字节。

学习阶段不需要过度关注它。

但要知道：

```text
SELECT * 会让每行返回更多字段，可能增加 IO、网络传输和内存成本。
```

所以后端接口中应该尽量只查需要的列。

---

## 七、Filter 和 Index Cond 的区别

### 1. Filter

```text
Filter: (email = 'user_50000@example.com'::text)
```

表示从表中读出行后，再过滤。

如果是 `Seq Scan`，通常就是一行一行读出来再判断条件。

### 2. Index Cond

```text
Index Cond: (email = 'user_50000@example.com'::text)
```

表示这个条件可以用于索引定位。

一般来说，`Index Cond` 比 `Filter` 更说明索引真正参与了查找。

但有时执行计划里可能同时出现：

```text
Index Cond: (...)
Filter: (...)
```

表示一部分条件用于索引定位，另一部分条件在取出行后继续过滤。

---

## 八、Sort

查询：

```sql
EXPLAIN
SELECT id, user_id, title, created_at
FROM demo.perf_posts
WHERE user_id = 100
ORDER BY created_at DESC
LIMIT 20;
```

如果没有合适的组合索引，可能出现：

```text
Sort
  Sort Key: created_at DESC
```

这表示 PostgreSQL 需要额外排序。

如果创建：

```sql
CREATE INDEX perf_posts_user_id_created_at_idx
ON demo.perf_posts (user_id, created_at DESC);
```

执行计划可能不再需要单独 `Sort`，因为索引顺序已经满足排序。

---

## 九、Limit

带分页或取前几条时：

```sql
SELECT ...
ORDER BY created_at DESC
LIMIT 20;
```

执行计划会出现：

```text
Limit
```

`LIMIT` 本身不一定慢。关键要看：

```text
数据库能不能用索引顺序快速拿到前 20 条。
```

如果必须先扫描大量数据、排序全部结果，再取 20 条，仍然会慢。

---

## 十、使用更易读的格式

可以使用：

```sql
EXPLAIN (FORMAT TEXT)
SELECT ...
```

默认就是文本格式。

也可以使用：

```sql
EXPLAIN (FORMAT JSON)
SELECT ...
```

JSON 格式适合程序分析，但初学时文本格式更直观。

常用：

```sql
EXPLAIN (COSTS, VERBOSE)
SELECT ...
```

学习阶段先用最简单的 `EXPLAIN` 即可。

---

## 十一、EXPLAIN 不会真正执行查询

普通 `EXPLAIN` 不会执行查询，所以：

```sql
EXPLAIN
UPDATE demo.perf_users
SET status = 'disabled'
WHERE id = 1;
```

只显示计划，不会真的更新数据。

但后面要学的：

```sql
EXPLAIN ANALYZE
```

会真正执行 SQL。

如果对 `UPDATE`、`DELETE`、`INSERT` 使用 `EXPLAIN ANALYZE`，一定要非常小心，必要时放进事务里然后回滚。

---

## 十二、看 EXPLAIN 的顺序

初学时可以按下面顺序看：

1. 最底层扫描了哪张表。
2. 是 `Seq Scan` 还是 `Index Scan`。
3. 条件是在 `Index Cond` 里，还是只在 `Filter` 里。
4. 估算返回多少行：`rows`。
5. 有没有额外 `Sort`。
6. 有没有 `Limit`，以及是否能配合索引提前停止。
7. 总成本大概是多少。

不要一上来就试图理解所有节点。先抓住最影响单表查询性能的几个点。

---

## 十三、常见误区

### 1. cost 是执行时间吗？

不是。

`cost` 是优化器内部估算单位，不是毫秒。

### 2. EXPLAIN 会执行 SQL 吗？

普通 `EXPLAIN` 不会。

`EXPLAIN ANALYZE` 会执行。

### 3. Seq Scan 一定不好吗？

不是。

小表、返回大量数据、选择性差时，顺序扫描可能是最优选择。

### 4. Index Cond 和 Filter 是一回事吗？

不是。

`Index Cond` 表示条件用于索引定位。`Filter` 表示读出行后再过滤。

---

## 十四、本节达标标准

学完本节后，你应该能够做到：

- 使用 `EXPLAIN` 查看执行计划。
- 知道普通 `EXPLAIN` 不会真正执行查询。
- 识别 `Seq Scan`、`Index Scan`、`Bitmap Index Scan`。
- 解释 `cost` 是估算成本，不是毫秒。
- 解释 `rows` 是估算返回行数。
- 区分 `Filter` 和 `Index Cond`。
- 识别查询中是否存在额外 `Sort`。
- 能按步骤初步阅读一个简单执行计划。
