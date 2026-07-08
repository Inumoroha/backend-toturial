# 5. 慢查询日志与 pg_stat_activity

本节目标：学会观察 PostgreSQL 正在执行什么，能初步定位慢查询、长事务和锁等待。

生产环境出问题时，不要只看应用日志。

数据库本身也要看。

最常用的入口是：

```text
pg_stat_activity
慢查询日志
EXPLAIN ANALYZE
```

---

## 一、pg_stat_activity 是什么

`pg_stat_activity` 是 PostgreSQL 提供的系统视图。

它可以看到当前数据库连接在做什么。

常用查询：

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS running_time,
    query
FROM pg_stat_activity
WHERE datname = current_database()
ORDER BY running_time DESC;
```

这能告诉你：

- 谁连着数据库。
- 当前连接状态。
- SQL 运行了多久。
- 是否正在等待锁或 IO。
- 正在执行什么 SQL。

---

## 二、state 怎么看

常见状态：

| state | 含义 |
| --- | --- |
| `active` | 正在执行 SQL |
| `idle` | 连接空闲 |
| `idle in transaction` | 事务打开但当前空闲 |
| `idle in transaction (aborted)` | 事务出错后未回滚 |

重点关注：

```text
active 时间很长。
idle in transaction 时间很长。
```

它们通常是线上问题的入口。

---

## 三、查运行时间最长的 SQL

```sql
SELECT
    pid,
    state,
    now() - query_start AS running_time,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE datname = current_database()
  AND state <> 'idle'
ORDER BY running_time DESC
LIMIT 20;
```

如果某条 SQL 跑了几十秒甚至几分钟，要进一步分析：

- 是否缺索引。
- 是否扫描了太多数据。
- 是否被锁阻塞。
- 是否返回了过多数据。
- 是否在事务里等待。

---

## 四、查 idle in transaction

```sql
SELECT
    pid,
    usename,
    client_addr,
    now() - xact_start AS transaction_age,
    query
FROM pg_stat_activity
WHERE datname = current_database()
  AND state = 'idle in transaction'
ORDER BY transaction_age DESC;
```

如果看到很老的事务，要警惕。

它可能：

- 持有锁。
- 阻止 VACUUM。
- 造成表膨胀。

应用代码要检查是否有事务忘记 `Commit` 或 `Rollback`。

---

## 五、查锁等待

先看哪些连接正在等锁：

```sql
SELECT
    pid,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS running_time,
    query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock'
ORDER BY running_time DESC;
```

如果有结果，说明有 SQL 正在等锁。

进一步可以查谁阻塞谁：

```sql
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks blocked_locks
  ON blocked_locks.pid = blocked.pid
JOIN pg_locks blocking_locks
  ON blocking_locks.locktype = blocked_locks.locktype
 AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
 AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
 AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
 AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
 AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
 AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
 AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
 AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
 AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
 AND blocking_locks.pid <> blocked_locks.pid
JOIN pg_stat_activity blocking
  ON blocking.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted
  AND blocking_locks.granted;
```

这条查询较长，但非常有用。

它可以告诉你：

```text
哪个连接被阻塞。
哪个连接正在阻塞别人。
```

---

## 六、慢查询日志是什么

`pg_stat_activity` 只能看当前正在发生的事情。

如果慢查询已经结束，就要靠日志。

PostgreSQL 可以配置：

```conf
log_min_duration_statement = 500ms
```

含义：

```text
执行时间超过 500ms 的 SQL 记录到日志。
```

具体配置方式取决于你的部署方式：

- 本地配置文件 `postgresql.conf`。
- Docker 环境变量或挂载配置。
- 云数据库控制台参数组。

---

## 七、慢查询阈值怎么设

学习环境可以设得低一点：

```text
100ms
```

生产环境要结合业务。

常见选择：

```text
200ms
500ms
1000ms
```

不要一开始设成 `0` 记录所有 SQL。

记录所有 SQL 会产生大量日志，对性能和存储都有影响。

---

## 八、拿到慢 SQL 后怎么做

流程：

```text
找到慢 SQL。
确认参数。
在测试环境构造类似数据量。
使用 EXPLAIN ANALYZE 分析。
判断是否缺索引或查询写法不合理。
修改 SQL 或索引。
再次 EXPLAIN ANALYZE 对比。
```

不要只凭感觉加索引。

一定要看执行计划。

---

## 九、短链接项目慢查询示例

如果列表查询慢：

```sql
SELECT id, code, original_url, visit_count, created_at, updated_at
FROM app.short_links
ORDER BY created_at DESC
LIMIT 20 OFFSET 50000;
```

可能原因：

- `created_at` 没有索引。
- `OFFSET` 太大。
- 返回字段太多。

分析：

```sql
EXPLAIN ANALYZE
SELECT id, code, original_url, visit_count, created_at, updated_at
FROM app.short_links
ORDER BY created_at DESC
LIMIT 20 OFFSET 50000;
```

优化方向：

- 添加或确认 `created_at DESC` 索引。
- 改成游标分页。
- 控制最大页数。

---

## 十、取消问题查询

如果某条查询明显有问题，可以取消：

```sql
SELECT pg_cancel_backend(pid);
```

如果取消无效，可以终止连接：

```sql
SELECT pg_terminate_backend(pid);
```

区别：

```text
pg_cancel_backend：请求取消当前查询。
pg_terminate_backend：断开整个连接。
```

生产使用要谨慎。

尤其是正在执行重要写入或迁移时，不要随便终止。

---

## 十一、常见排查顺序

接口变慢时：

1. 看应用日志，确认哪个接口慢。
2. 查 `pg_stat_activity`，看是否有长 SQL。
3. 看是否有锁等待。
4. 找到慢 SQL。
5. 用 `EXPLAIN ANALYZE` 分析。
6. 检查索引。
7. 检查连接池是否耗尽。
8. 检查最近是否上线迁移或新版本。

这个顺序不一定固定，但能避免盲目猜。

---

## 十二、本节达标标准

学完本节后，你应该能够做到：

- 使用 `pg_stat_activity` 查看当前连接。
- 找出运行时间最长的 SQL。
- 找出 `idle in transaction`。
- 判断连接是否在等待锁。
- 知道慢查询日志的作用。
- 知道 `log_min_duration_statement` 的含义。
- 能用 `EXPLAIN ANALYZE` 分析慢 SQL。
- 谨慎使用 `pg_cancel_backend` 和 `pg_terminate_backend`。

