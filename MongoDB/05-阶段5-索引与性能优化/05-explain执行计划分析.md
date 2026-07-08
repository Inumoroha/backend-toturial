# 05. explain 执行计划分析

索引优化不能靠猜。`explain("executionStats")` 是判断查询是否健康的核心工具。

## 1. 基本用法

```javascript
db.orders.find({
  user_id: user1._id
}).explain("executionStats")
```

也可以对排序查询：

```javascript
db.orders.find({
  user_id: user1._id
}).sort({ created_at: -1 }).explain("executionStats")
```

## 2. 重点看哪些字段

常看：

```text
winningPlan
executionStats.nReturned
executionStats.totalKeysExamined
executionStats.totalDocsExamined
executionStats.executionTimeMillis
```

含义：

| 字段 | 含义 |
| --- | --- |
| `winningPlan` | 最终选择的执行计划 |
| `nReturned` | 返回文档数 |
| `totalKeysExamined` | 扫描索引键数量 |
| `totalDocsExamined` | 扫描文档数量 |
| `executionTimeMillis` | 执行耗时 |

## 3. COLLSCAN

如果看到：

```text
stage: "COLLSCAN"
```

表示集合扫描。

例子：

```javascript
db.orders.find({
  user_id: user1._id
}).explain("executionStats")
```

如果没有 `user_id` 索引，可能看到 COLLSCAN。

这通常是需要优化的信号。

## 4. IXSCAN

创建索引：

```javascript
db.orders.createIndex({ user_id: 1, created_at: -1 })
```

再看：

```javascript
db.orders.find({
  user_id: user1._id
}).sort({ created_at: -1 }).explain("executionStats")
```

如果看到：

```text
stage: "IXSCAN"
```

表示使用了索引扫描。

注意：有 IXSCAN 不代表一定快。还要看扫描了多少 key 和 doc。

## 5. nReturned 和 totalDocsExamined

健康查询通常希望：

```text
totalDocsExamined 接近 nReturned
```

例如：

```text
nReturned: 20
totalDocsExamined: 20
```

如果：

```text
nReturned: 20
totalDocsExamined: 15000
```

说明扫描了大量无关文档，索引可能不合适。

## 6. totalKeysExamined

如果：

```text
totalKeysExamined 很大
totalDocsExamined 较小
```

可能说明：

- 索引能覆盖部分过滤。
- 但范围太大。
- 或索引选择性不够好。

这时要结合查询条件、字段基数、排序和 limit 分析。

## 7. SORT 阶段

如果执行计划里出现单独的：

```text
stage: "SORT"
```

表示 MongoDB 可能需要内存排序。

如果排序数据量大，会变慢。

可以考虑把排序字段放入复合索引：

```javascript
db.orders.createIndex({ user_id: 1, status: 1, created_at: -1 })
```

支持：

```javascript
db.orders.find({
  user_id: user1._id,
  status: "paid"
}).sort({ created_at: -1 })
```

## 8. 对比优化前后

优化前：

```javascript
db.orders.find({
  user_id: user1._id,
  status: "paid"
}).sort({ created_at: -1 }).explain("executionStats")
```

创建索引：

```javascript
db.orders.createIndex({ user_id: 1, status: 1, created_at: -1 })
```

优化后再跑同样 explain。

比较：

```text
stage
nReturned
totalKeysExamined
totalDocsExamined
executionTimeMillis
```

不要只看耗时。学习环境数据少、缓存影响大，耗时可能不稳定。扫描量更能说明问题。

## 9. 聚合 explain

聚合也可以 explain：

```javascript
db.orders.explain("executionStats").aggregate([
  { $match: { status: "paid" } },
  { $group: { _id: "$user_id", total: { $sum: "$total_amount" } } }
])
```

重点看 `$match` 是否利用索引。

聚合优化原则：

- `$match` 尽量前置。
- `$sort` 尽量被索引支持。
- `$lookup` 前缩小数据量。

## 10. 本节练习

完成：

1. 删除 orders 非 `_id` 索引。
2. 对用户订单查询跑 explain。
3. 记录是否 COLLSCAN。
4. 创建 `{ user_id: 1, created_at: -1 }`。
5. 再跑 explain。
6. 对比 `totalDocsExamined`。
7. 对用户 + 状态 + 时间排序查询跑 explain。
8. 创建 `{ user_id: 1, status: 1, created_at: -1 }`。
9. 再对比执行计划。

## 11. 本节小结

你需要记住：

- `explain("executionStats")` 是索引优化核心工具。
- `COLLSCAN` 通常表示集合扫描。
- `IXSCAN` 表示使用索引扫描。
- `nReturned` 是返回数量。
- `totalDocsExamined` 是扫描文档数量。
- `totalDocsExamined` 远大于 `nReturned` 通常需要优化。
- 有 IXSCAN 不代表一定健康，还要看扫描量。

