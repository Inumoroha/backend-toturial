# 第 4 阶段：聚合管道

聚合管道是 MongoDB 做统计、报表、数据转换和多阶段查询的核心能力。前面你已经能完成 CRUD 和文档建模，本阶段要学会把数据“加工”成业务需要的结果，例如订单金额按天统计、商品销量排行榜、用户消费排行、文章评论数统计。

## 本阶段目标

完成本阶段后，你应该能做到：

- 理解 aggregation pipeline 的执行模型。
- 熟练使用 `$match`、`$project`、`$group`、`$sort`、`$limit`、`$skip`。
- 使用 `$addFields` / `$set` 计算新字段。
- 使用 `$unwind` 展开数组。
- 使用 `$lookup` 做集合关联，并理解它的边界。
- 使用日期表达式做按天、按月统计。
- 在 Go Driver v2 中构造和执行 aggregation pipeline。
- 独立完成订单统计、销量排行、用户消费排行、文章数据统计。

## 学习文件顺序

| 顺序 | 文件 | 作用 |
| --- | --- | --- |
| 00 | `00-README.md` | 本阶段总览 |
| 01 | `01-聚合管道基本思想.md` | 理解 pipeline 和 stage |
| 02 | `02-准备练习数据.md` | 初始化订单、商品、用户、文章数据 |
| 03 | `03-match-project-sort-limit-skip.md` | 学基础过滤、投影、排序、分页 |
| 04 | `04-group分组统计.md` | 学 `$group`、`$sum`、`$avg`、`$min`、`$max` |
| 05 | `05-addFields-set字段计算.md` | 学计算字段和字段转换 |
| 06 | `06-unwind数组展开.md` | 学处理订单 items、标签、评论等数组 |
| 07 | `07-lookup集合关联.md` | 学集合关联和 `$lookup` 使用边界 |
| 08 | `08-时间统计与日期表达式.md` | 学按日、按月、最近 30 天统计 |
| 09 | `09-Go中构造AggregationPipeline.md` | 学 Go Driver v2 中执行聚合 |
| 10 | `10-阶段练习-订单与内容统计.md` | 综合练习 |
| 11 | `11-阶段验收与聚合速查.md` | 验收和命令速查 |

## 本阶段数据库

统一使用：

```javascript
use aggregation_stage4
```

本阶段会创建：

```text
users
products
orders
posts
comments
events
```

## 聚合管道一句话理解

```text
集合中的文档
  -> 第 1 个 stage 过滤或转换
  -> 第 2 个 stage 分组或计算
  -> 第 3 个 stage 排序或限制
  -> 输出最终结果
```

例如：

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  { $group: { _id: "$user_id", total_amount: { $sum: "$total_amount" } } },
  { $sort: { total_amount: -1 } }
])
```

含义：

1. 只保留已支付订单。
2. 按用户分组，统计消费总额。
3. 按消费总额降序排序。

## 官方资料

本阶段主要参考：

- Aggregation Pipeline：<https://www.mongodb.com/docs/manual/core/aggregation-pipeline/>
- Aggregation Stages：<https://www.mongodb.com/docs/manual/reference/operator/aggregation-pipeline/>
- `$group`：<https://www.mongodb.com/docs/manual/reference/operator/aggregation/group/>
- `$lookup`：<https://www.mongodb.com/docs/manual/reference/operator/aggregation/lookup/>
- Go Driver Aggregation：<https://www.mongodb.com/docs/drivers/go/current/aggregation/>

