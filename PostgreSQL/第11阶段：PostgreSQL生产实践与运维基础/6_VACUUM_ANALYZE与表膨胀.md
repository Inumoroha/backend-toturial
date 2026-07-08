# 6. VACUUM、ANALYZE 与表膨胀

本节目标：理解 PostgreSQL 为什么需要 VACUUM 和 ANALYZE，知道长事务、频繁更新和表膨胀之间的关系。

PostgreSQL 使用 MVCC 保证并发读写。

MVCC 带来的一个结果是：

```text
UPDATE 和 DELETE 不会立刻物理移除旧数据版本。
```

这些旧版本后续需要清理。

清理工作主要依赖：

```text
VACUUM
autovacuum
```

---

## 一、MVCC 和旧版本

执行：

```sql
UPDATE app.short_links
SET original_url = 'https://example.com/new'
WHERE code = 'abc123';
```

PostgreSQL 不是简单地原地覆盖旧行。

可以粗略理解为：

```text
旧版本保留。
新版本写入。
不同事务根据自己的快照看到不同版本。
```

这样读写可以更好并发。

但旧版本不能永远留着。

否则表会越来越大。

---

## 二、VACUUM 做什么

`VACUUM` 的作用是：

```text
清理不再被任何事务需要的旧行版本。
让空间可以被后续写入复用。
维护可见性信息。
防止事务 ID 回卷风险。
```

普通 `VACUUM` 通常不会把磁盘文件立刻缩小。

它更多是让表内部空间可以复用。

---

## 三、ANALYZE 做什么

`ANALYZE` 的作用是：

```text
收集表的统计信息。
帮助查询优化器选择执行计划。
```

例如优化器需要知道：

- 表大概有多少行。
- 某列值分布如何。
- 某个条件选择性高不高。

如果统计信息过旧，优化器可能误判，选择错误计划。

例如该走索引时走了顺序扫描，或者反过来。

---

## 四、autovacuum 是什么

PostgreSQL 默认有自动清理机制：

```text
autovacuum
```

它会自动对表执行：

```text
VACUUM
ANALYZE
```

所以大多数时候你不需要手动频繁执行。

但你需要知道：

```text
autovacuum 不是魔法。
长事务可能阻止它清理旧版本。
高写入表可能需要调整参数。
```

---

## 五、长事务为什么危险

如果一个事务很久不结束：

```sql
BEGIN;
SELECT * FROM app.short_links;
-- 很久不 COMMIT，也不 ROLLBACK
```

PostgreSQL 需要保留这个事务可能看到的旧版本。

这会导致：

```text
VACUUM 不能清理某些旧数据。
表和索引越来越大。
查询变慢。
磁盘占用增加。
```

这就是为什么 `idle in transaction` 很危险。

---

## 六、表膨胀是什么

表膨胀可以简单理解为：

```text
表文件里有很多已经没用但还占空间的旧行版本。
```

常见原因：

- 高频 UPDATE。
- 高频 DELETE。
- 长事务阻止清理。
- autovacuum 跟不上。
- 大批量数据变更后没有及时维护。

短链接项目里，如果每次跳转都更新：

```sql
visit_count = visit_count + 1
```

热门短链会产生大量行版本。

第一版可以接受，但要知道高访问量下可能带来膨胀和写入热点。

---

## 七、手动执行 VACUUM 和 ANALYZE

手动清理：

```sql
VACUUM app.short_links;
```

手动分析：

```sql
ANALYZE app.short_links;
```

一起执行：

```sql
VACUUM ANALYZE app.short_links;
```

一般用于：

- 大量导入数据后。
- 大量删除或更新后。
- 查询计划明显不合理时。
- 测试环境观察效果。

生产上不要盲目频繁手动执行，先理解原因。

---

## 八、VACUUM FULL 要谨慎

`VACUUM FULL` 会重写表，可以把磁盘空间还给操作系统。

```sql
VACUUM FULL app.short_links;
```

但它会持有较强的锁。

生产大表上执行可能阻塞业务。

所以不要把 `VACUUM FULL` 当作日常维护命令。

它更像是：

```text
确认需要回收磁盘空间，并安排维护窗口后使用的操作。
```

---

## 九、查看表统计信息

可以看表的活动统计：

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE schemaname = 'app'
ORDER BY n_dead_tup DESC;
```

字段含义：

| 字段 | 含义 |
| --- | --- |
| `n_live_tup` | 估算活跃行数 |
| `n_dead_tup` | 估算死元组数量 |
| `last_vacuum` | 上次手动 vacuum |
| `last_autovacuum` | 上次自动 vacuum |
| `last_analyze` | 上次手动 analyze |
| `last_autoanalyze` | 上次自动 analyze |

如果 `n_dead_tup` 很高，并且很久没有 autovacuum，就要关注。

---

## 十、查看表和索引大小

```sql
SELECT
    relname,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
    pg_size_pretty(pg_relation_size(relid)) AS table_size,
    pg_size_pretty(pg_indexes_size(relid)) AS indexes_size
FROM pg_catalog.pg_statio_user_tables
WHERE schemaname = 'app'
ORDER BY pg_total_relation_size(relid) DESC;
```

这能帮助你观察：

- 哪些表最大。
- 表数据大还是索引大。
- 是否有异常增长。

---

## 十一、短链接项目的特殊点

短链接服务中：

```sql
UPDATE app.short_links
SET visit_count = visit_count + 1
WHERE code = $1;
```

如果访问量很高，这张表会频繁更新。

可能问题：

- 热门行锁竞争。
- 旧版本增加。
- autovacuum 压力增加。
- 索引维护成本增加。

后续优化方向：

- Redis 计数，定期回写。
- 按天统计访问量。
- 访问日志异步写入。
- 对热门链接做批量聚合。

第一版先不用过早优化，但要有这个意识。

---

## 十二、本节达标标准

学完本节后，你应该能够做到：

- 解释 PostgreSQL 为什么需要 VACUUM。
- 解释 ANALYZE 和查询计划的关系。
- 理解 autovacuum 的作用。
- 知道长事务会阻止旧版本清理。
- 知道表膨胀的基本原因。
- 谨慎使用 `VACUUM FULL`。
- 使用 `pg_stat_user_tables` 查看死元组和 vacuum 时间。
- 使用 SQL 查看表和索引大小。

