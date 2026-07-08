# 8. Seq Scan、Index Scan、Bitmap Index Scan

本节目标：理解 PostgreSQL 执行计划中三种常见扫描方式：`Seq Scan`、`Index Scan`、`Bitmap Index Scan`，知道它们分别适合什么场景。

当你看执行计划时，最常见的问题之一是：

```text
这条 SQL 到底是怎么从表里找到数据的？
```

扫描方式就是答案。

---

## 一、Seq Scan：顺序扫描

`Seq Scan` 是 Sequential Scan，顺序扫描。

它表示：

```text
从表中顺序读取数据，一行一行检查是否满足条件。
```

例子：

```sql
EXPLAIN
SELECT id, username, email
FROM demo.perf_users
WHERE status = 'active';
```

可能看到：

```text
Seq Scan on perf_users
  Filter: (status = 'active'::text)
```

### Seq Scan 什么时候正常？

#### 1. 小表

表只有几十行或几百行时，顺序扫描很正常。

走索引还要先读索引，再读表，可能不如直接扫表。

#### 2. 返回大量数据

如果查询返回表中大部分行：

```sql
WHERE status = 'active'
```

而大多数用户都是 `active`，顺序扫描可能更快。

#### 3. 没有合适索引

如果条件字段没有索引，通常只能顺序扫描。

### Seq Scan 什么时候需要警惕？

如果表很大，查询只需要少量行，却出现：

```text
Seq Scan
Rows Removed by Filter: 很多
```

就要考虑是否缺少索引，或者索引没有被有效使用。

---

## 二、Index Scan：索引扫描

`Index Scan` 表示 PostgreSQL 使用索引定位数据。

例子：

```sql
CREATE INDEX perf_users_email_idx
ON demo.perf_users (email);

EXPLAIN
SELECT id, username, email
FROM demo.perf_users
WHERE email = 'user_50000@example.com';
```

可能看到：

```text
Index Scan using perf_users_email_idx on perf_users
  Index Cond: (email = 'user_50000@example.com'::text)
```

### Index Scan 的基本过程

可以粗略理解为：

```text
1. 在索引中找到符合条件的键。
2. 根据索引记录的位置，到表中取出完整行。
```

所以 Index Scan 很适合：

- 返回少量行。
- 条件选择性好。
- 按索引顺序排序并 `LIMIT`。

### Index Scan 不一定适合大量结果

如果通过索引找到大量行，还要频繁回表读取，随机访问成本可能比较高。

这时 PostgreSQL 可能选择：

- `Seq Scan`
- `Bitmap Index Scan` + `Bitmap Heap Scan`

---

## 三、Index Only Scan

有时你会看到：

```text
Index Only Scan
```

它表示 PostgreSQL 尝试只通过索引返回查询结果，不回表或少回表。

例如索引包含查询所需字段：

```sql
CREATE INDEX perf_users_email_username_idx
ON demo.perf_users (email, username);
```

查询：

```sql
EXPLAIN
SELECT email, username
FROM demo.perf_users
WHERE email = 'user_50000@example.com';
```

可能出现 `Index Only Scan`。

注意：因为 MVCC 可见性判断，`Index Only Scan` 是否真的完全不访问表，还和可见性映射有关。初学时先知道它是“尽量只读索引”的扫描方式即可。

---

## 四、Bitmap Index Scan：位图索引扫描

`Bitmap Index Scan` 通常不会单独出现，而是配合：

```text
Bitmap Heap Scan
```

典型结构：

```text
Bitmap Heap Scan on perf_orders
  Recheck Cond: (status = 'pending'::text)
  -> Bitmap Index Scan on perf_orders_status_idx
       Index Cond: (status = 'pending'::text)
```

它的基本思路是：

```text
1. 先通过索引找到一批符合条件的行位置。
2. 把这些位置整理成位图。
3. 再按数据页批量访问表。
```

### 为什么需要 Bitmap 扫描？

普通 Index Scan 找一行取一行，适合少量结果。

如果结果不止几行，而是一批数据，逐条回表可能随机 IO 较多。

Bitmap 扫描会先收集行位置，再更有组织地访问表页，可能更划算。

---

## 五、Bitmap 扫描适合什么场景

### 1. 返回中等数量的数据

不是少到适合简单 Index Scan，也不是多到适合 Seq Scan。

例如返回几千行、几万行，具体取决于表大小、缓存、行宽等。

### 2. 多个索引条件组合

PostgreSQL 可以使用 BitmapAnd 或 BitmapOr 合并多个索引结果。

例如：

```sql
WHERE status = 'pending'
  AND user_id = 12345
```

如果有两个单列索引：

```sql
CREATE INDEX perf_orders_status_idx
ON demo.perf_orders (status);

CREATE INDEX perf_orders_user_id_idx
ON demo.perf_orders (user_id);
```

执行计划有时可能用 BitmapAnd 把两个索引结果合起来。

但这不代表单列索引一定优于组合索引。

对于稳定高频查询：

```sql
WHERE user_id = ?
  AND status = ?
ORDER BY paid_at DESC
LIMIT ?
```

贴合查询的组合索引通常更值得考虑。

---

## 六、三种扫描方式对比

| 扫描方式 | 直觉 | 适合场景 |
| --- | --- | --- |
| `Seq Scan` | 从头到尾扫表 | 小表、返回大量数据、没有合适索引 |
| `Index Scan` | 先查索引，再取表数据 | 返回少量数据、选择性好、排序 + LIMIT |
| `Bitmap Index Scan` | 先用索引收集一批位置，再批量读表 | 返回中等数量数据、多个索引条件组合 |

不要机械地认为：

```text
Index Scan 一定最好。
Seq Scan 一定最差。
Bitmap 一定复杂。
```

执行计划是优化器根据成本选择的结果。真正要看的是它是否符合数据量和业务查询模式。

---

## 七、Rows Removed by Filter

使用 `EXPLAIN ANALYZE` 时，可能看到：

```text
Rows Removed by Filter: 99999
```

这表示扫描过程中有很多行被过滤掉。

例如：

```sql
EXPLAIN ANALYZE
SELECT id, username, email
FROM demo.perf_users
WHERE email = 'user_50000@example.com';
```

如果没有 email 索引，可能扫描很多行，最后只保留 1 行。

这类情况通常说明：

```text
查询条件选择性很好，但缺少能用于定位的索引。
```

---

## 八、Heap Blocks

在 Bitmap 扫描中，使用 `BUFFERS` 或详细计划时可能看到和 heap block 相关的信息。

初学时可以粗略理解：

```text
Heap 表示表数据本身。
Index 表示索引数据。
Bitmap Heap Scan 表示根据位图去访问表数据页。
```

当前阶段不需要深入页结构，但要知道：

```text
索引扫描通常不只是读索引，还可能需要访问表数据。
```

---

## 九、如何根据扫描方式判断优化方向

### 1. 大表少量结果却 Seq Scan

可能需要索引，或 SQL 写法导致索引无法使用。

检查：

- WHERE 字段有没有索引。
- 是否在字段上使用了函数。
- 是否有类型转换。
- LIKE 是否有前导 `%`。
- 组合索引字段顺序是否匹配。

### 2. Index Scan 但仍然慢

可能原因：

- 返回行数比预期多。
- 回表次数太多。
- 索引选择性不够。
- 还需要额外排序。

可以考虑：

- 组合索引。
- 覆盖索引或 `INCLUDE`。
- 部分索引。
- 改写查询条件。

### 3. Bitmap Scan 但仍然慢

可能原因：

- 返回数据量仍然太大。
- 单列索引组合效果不如组合索引。
- 查询条件选择性差。
- 后续排序或聚合很重。

---

## 十、常见误区

### 1. Seq Scan 一定要优化掉？

不是。

小表或大量返回时，Seq Scan 很正常。

### 2. Index Scan 一定最快？

不是。

如果返回大量行，Index Scan 可能比 Seq Scan 更慢。

### 3. Bitmap Index Scan 是索引失效吗？

不是。

它也是使用索引的一种方式，只是访问表数据的策略不同。

### 4. 扫描方式固定不变吗？

不是。

数据量变化、统计信息变化、参数值不同、缓存状态不同，都可能让 PostgreSQL 选择不同计划。

---

## 十一、本节达标标准

学完本节后，你应该能够做到：

- 解释 `Seq Scan` 是顺序扫描。
- 解释 `Index Scan` 是使用索引定位数据。
- 知道 `Index Only Scan` 尝试只从索引返回结果。
- 解释 `Bitmap Index Scan` 和 `Bitmap Heap Scan` 的配合关系。
- 知道三种扫描方式分别适合什么场景。
- 不再简单认为 `Seq Scan` 一定坏、`Index Scan` 一定好。
- 能根据扫描方式初步判断优化方向。
