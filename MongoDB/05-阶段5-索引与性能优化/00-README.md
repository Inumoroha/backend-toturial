# 第 5 阶段：索引与性能优化

索引是 MongoDB 从“能查”走向“查得稳、查得快”的关键。本阶段重点学习索引类型、复合索引字段顺序、ESR 原则、`explain("executionStats")`、慢查询分析、覆盖索引、分页优化和批量写入。

## 本阶段目标

完成本阶段后，你应该能做到：

- 理解为什么没有索引会出现 collection scan。
- 创建单字段索引、复合索引、唯一索引、TTL 索引、部分索引、文本索引。
- 根据查询条件、排序和范围过滤设计复合索引。
- 理解 ESR：Equality、Sort、Range。
- 使用 `explain("executionStats")` 分析查询。
- 看懂 `COLLSCAN`、`IXSCAN`、`nReturned`、`totalDocsExamined`。
- 识别慢查询和低效索引。
- 理解覆盖索引。
- 把 `skip + limit` 深分页改成游标分页。
- 在 Go Driver v2 中使用索引和 `BulkWrite`。
- 理解索引不是越多越好，写入也有成本。

## 学习文件顺序

| 顺序 | 文件 | 作用 |
| --- | --- | --- |
| 00 | `00-README.md` | 本阶段总览 |
| 01 | `01-索引基本原理与COLLSCAN.md` | 理解索引为什么能加速查询 |
| 02 | `02-准备性能练习数据.md` | 准备订单、用户、token、文章数据 |
| 03 | `03-单字段唯一TTL部分文本索引.md` | 学常见索引类型 |
| 04 | `04-复合索引与ESR原则.md` | 学复合索引字段顺序 |
| 05 | `05-explain执行计划分析.md` | 学执行计划和核心指标 |
| 06 | `06-慢查询与索引优化流程.md` | 学慢查询定位和优化步骤 |
| 07 | `07-覆盖索引与投影优化.md` | 学 covered query 和字段裁剪 |
| 08 | `08-分页优化-从skip到游标分页.md` | 优化深分页 |
| 09 | `09-写入成本与BulkWrite.md` | 学索引写入成本和批量写入 |
| 10 | `10-Go中创建索引与性能查询.md` | Go Driver v2 中创建索引和批量写 |
| 11 | `11-阶段练习-订单查询优化.md` | 综合练习 |
| 12 | `12-阶段验收与索引评审清单.md` | 验收和评审清单 |

## 本阶段数据库

统一使用：

```javascript
use performance_stage5
```

## 本阶段核心原则

```text
先明确查询模式，再设计索引。
先用 explain 证明问题，再说优化有效。
索引服务于查询，不服务于字段数量。
读性能和写性能要一起评估。
```

## 官方资料

本阶段主要参考：

- Indexes：<https://www.mongodb.com/docs/manual/indexes/>
- Compound Indexes：<https://www.mongodb.com/docs/manual/core/indexes/index-types/index-compound/>
- ESR Guideline：<https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/>
- Explain Results：<https://www.mongodb.com/docs/manual/reference/explain-results/>
- TTL Indexes：<https://www.mongodb.com/docs/manual/core/index-ttl/>
- Partial Indexes：<https://www.mongodb.com/docs/manual/core/index-partial/>
- Text Indexes：<https://www.mongodb.com/docs/manual/core/indexes/index-types/index-text/>
- Go Driver Indexes：<https://www.mongodb.com/docs/drivers/go/current/indexes/>
- Go Driver Bulk Write：<https://www.mongodb.com/docs/drivers/go/current/usage-examples/bulkWrite/>

