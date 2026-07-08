# 7. EXPLAIN ANALYZE：真实执行分析

本节目标：掌握 `EXPLAIN ANALYZE` 的作用，能看懂真实执行时间、真实行数、循环次数，并知道如何安全地分析写操作。

`EXPLAIN` 显示的是 PostgreSQL 的计划。

`EXPLAIN ANALYZE` 会真正执行 SQL，并显示真实执行情况。

```sql
EXPLAIN ANALYZE
SELECT id, username, email
FROM demo.perf_users
WHERE email = 'user_50000@example.com';
```

可以理解为：

```text
EXPLAIN：数据库准备怎么做。
EXPLAIN ANALYZE：数据库实际做了什么，花了多久。
```

---

## 一、为什么需要 EXPLAIN ANALYZE

普通 `EXPLAIN` 只显示估算：

- 估算成本。
- 估算行数。
- 计划选择。

但慢查询分析时，我们还需要知道真实情况：

- 实际返回多少行。
- 每个步骤实际耗时多久。
- 某个节点执行了几次。
- 估算行数和真实行数是否差距很大。

这些要靠 `EXPLAIN ANALYZE`。

---

## 二、基本语法

```sql
EXPLAIN ANALYZE
SELECT ...
```

推荐常用写法：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, username, email
FROM demo.perf_users
WHERE email = 'user_50000@example.com';
```

`BUFFERS` 会显示缓存和磁盘读取相关信息，对性能分析很有帮助。

初学时如果觉得输出太多，可以先用：

```sql
EXPLAIN ANALYZE
SELECT ...
```

---

## 三、actual time

输出中常见：

```text
actual time=0.030..0.035
```

表示真实执行时间：

```text
开始输出第一行的时间..输出全部行的时间
```

单位是毫秒。

和 `cost` 不同：

```text
cost 是估算单位，不是毫秒。
actual time 是真实毫秒时间。
```

---

## 四、actual rows

输出中常见：

```text
rows=1
```

在 `EXPLAIN ANALYZE` 里，它表示实际返回行数。

例如：

```text
Index Scan using perf_users_email_idx on perf_users
  (cost=0.42..8.44 rows=1 width=48)
  (actual time=0.030..0.035 rows=1 loops=1)
```

这里有两种 `rows`：

- `cost` 那行里的 `rows=1`：估算行数。
- `actual` 那行里的 `rows=1`：真实行数。

如果估算行数和真实行数差距很大，优化器可能选错计划。

---

## 五、loops

```text
loops=1
```

表示这个节点执行了几次。

简单单表查询通常是 `loops=1`。

在 JOIN 或子查询中，某些节点可能执行很多次。

例如嵌套循环中，内层查询可能被外层每一行触发一次：

```text
loops=10000
```

这时即使单次执行很快，总成本也可能很高。

分析时要注意：

```text
总耗时大约和 actual time、rows、loops 都有关。
```

---

## 六、Planning Time 和 Execution Time

输出底部常见：

```text
Planning Time: 0.200 ms
Execution Time: 0.080 ms
```

含义：

- `Planning Time`：优化器生成执行计划花的时间。
- `Execution Time`：真正执行 SQL 花的时间。

大多数普通慢查询重点看 `Execution Time`。

如果 SQL 极其复杂、表和索引很多，也可能规划时间明显变长。

---

## 七、BUFFERS 怎么看

使用：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...
```

可能看到：

```text
Buffers: shared hit=10 read=2
```

粗略理解：

- `shared hit`：从 PostgreSQL 共享缓存中读到的页。
- `shared read`：需要从磁盘或操作系统缓存读取的页。

初学时不用过度深挖 Buffer 机制，但可以先建立直觉：

```text
扫描的数据页越多，IO 成本通常越高。
```

如果一个查询返回很少数据，却读了大量 buffer，就要怀疑是否扫描太多无关数据。

---

## 八、估算行数和真实行数差距

这是分析执行计划时非常重要的一点。

例如：

```text
cost 行：rows=10
actual 行：rows=100000
```

说明 PostgreSQL 以为只会返回 10 行，实际返回了 100000 行。

这可能导致它选择不合适的计划。

常见原因：

- 统计信息过旧。
- 数据分布极度不均匀。
- 多字段之间有相关性，但优化器估算不准。
- 查询条件写法复杂。

可以先尝试：

```sql
ANALYZE demo.perf_orders;
```

然后重新执行 `EXPLAIN ANALYZE` 对比。

---

## 九、安全分析 UPDATE、DELETE、INSERT

重要提醒：

```text
EXPLAIN ANALYZE 会真正执行 SQL。
```

如果你执行：

```sql
EXPLAIN ANALYZE
DELETE FROM demo.perf_orders
WHERE status = 'cancelled';
```

它会真的删除数据。

学习或排查时可以放进事务，然后回滚：

```sql
BEGIN;

EXPLAIN ANALYZE
DELETE FROM demo.perf_orders
WHERE status = 'cancelled';

ROLLBACK;
```

这样可以看到真实执行计划，同时撤销数据修改。

生产环境仍然要谨慎，因为即使回滚，语句执行过程中也可能持锁、产生 IO、影响系统。

---

## 十、对比优化前后

慢查询优化不要只看感觉，要对比数据。

优化前：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, order_no, user_id, created_at
FROM demo.perf_orders
WHERE status = 'pending'
ORDER BY created_at ASC
LIMIT 100;
```

创建索引：

```sql
CREATE INDEX perf_orders_pending_created_at_idx
ON demo.perf_orders (created_at ASC)
WHERE status = 'pending';

ANALYZE demo.perf_orders;
```

优化后：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, order_no, user_id, created_at
FROM demo.perf_orders
WHERE status = 'pending'
ORDER BY created_at ASC
LIMIT 100;
```

对比：

- 扫描方式是否变化。
- 是否少了 `Sort`。
- `Execution Time` 是否下降。
- `Buffers` 是否减少。
- 实际扫描行数是否减少。

---

## 十一、不要只看总时间

总时间很重要，但还要看慢在哪里。

例如：

```text
Seq Scan 扫了 500000 行，过滤后只剩 1 行。
```

说明过滤条件缺少有效索引。

又例如：

```text
扫描不慢，但 Sort 很慢。
```

说明可能需要能支持排序的组合索引，或者减少排序数据量。

再例如：

```text
Nested Loop 的内层 loops 很大。
```

说明 JOIN 方式或关联字段索引可能有问题。

当前阶段先重点掌握单表查询，后续遇到复杂 JOIN 时再深入。

---

## 十二、常见误区

### 1. EXPLAIN ANALYZE 只是分析，不会执行？

会执行。

这点非常重要。对写操作使用时必须谨慎。

### 2. actual time 一定每次都一样？

不是。

缓存状态、系统负载、数据量、并发情况都会影响实际时间。

### 3. 只看 Execution Time 就够了吗？

不够。

还要看扫描方式、实际行数、loops、Buffers、Sort 等节点，才能知道慢在哪里。

### 4. 优化后快一次就说明一定成功？

不一定。

最好多运行几次，并结合实际业务数据和查询频率判断。

---

## 十三、本节达标标准

学完本节后，你应该能够做到：

- 使用 `EXPLAIN ANALYZE` 查看真实执行情况。
- 知道 `EXPLAIN ANALYZE` 会真正执行 SQL。
- 解释 `actual time` 是真实毫秒时间。
- 区分估算 `rows` 和真实 `rows`。
- 理解 `loops` 的含义。
- 查看 `Planning Time` 和 `Execution Time`。
- 使用 `EXPLAIN (ANALYZE, BUFFERS)` 辅助分析 IO。
- 知道如何用事务安全分析写操作。
- 能对比优化前后的执行计划。
