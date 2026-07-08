# 03. match、project、sort、limit、skip

本节学习最基础的一组 stage：过滤、投影、排序、限制和跳过。它们经常出现在所有聚合管道里。

## 1. $match

`$match` 用于过滤文档，语法和 `find` 的过滤条件很像。

查询已支付订单：

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } }
])
```

查询 2026-07-02 之后支付的订单：

```javascript
db.orders.aggregate([
  {
    $match: {
      status: "paid",
      paid_at: { $gte: new Date("2026-07-02T00:00:00Z") }
    }
  }
])
```

建议：能放到 `$match` 的过滤尽量早放，减少后续 stage 处理的数据量。

## 2. $project

`$project` 控制输出字段，也可以重命名或计算字段。

只输出订单号、状态、金额：

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  {
    $project: {
      _id: 0,
      order_no: 1,
      status: 1,
      total_amount: 1
    }
  }
])
```

重命名字段：

```javascript
db.orders.aggregate([
  {
    $project: {
      _id: 0,
      order_no: 1,
      amount: "$total_amount"
    }
  }
])
```

## 3. $project 中计算字段

把金额从分转成元：

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  {
    $project: {
      _id: 0,
      order_no: 1,
      total_amount: 1,
      total_yuan: { $divide: ["$total_amount", 100] }
    }
  }
])
```

拼接字段：

```javascript
db.users.aggregate([
  {
    $project: {
      _id: 0,
      label: { $concat: ["$name", " <", "$email", ">"] }
    }
  }
])
```

## 4. $sort

按金额倒序：

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  { $sort: { total_amount: -1 } }
])
```

按状态升序、创建时间倒序：

```javascript
db.orders.aggregate([
  { $sort: { status: 1, created_at: -1 } }
])
```

如果排序字段有合适索引，性能会更好。后面第 5 阶段详细讲。

## 5. $limit

金额最高的 3 个已支付订单：

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  { $sort: { total_amount: -1 } },
  { $limit: 3 }
])
```

`$limit` 常和 `$sort` 搭配实现 Top N。

## 6. $skip

跳过前 2 条：

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  { $sort: { created_at: -1 } },
  { $skip: 2 },
  { $limit: 2 }
])
```

这类似分页。

注意：深分页时 `$skip` 仍然有性能问题。后面索引和分页优化阶段会处理。

## 7. Stage 顺序很重要

推荐：

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  { $sort: { created_at: -1 } },
  { $limit: 10 },
  { $project: { _id: 0, order_no: 1, total_amount: 1, created_at: 1 } }
])
```

不推荐：

```javascript
db.orders.aggregate([
  { $project: { order_no: 1, total_amount: 1, status: 1, created_at: 1 } },
  { $match: { status: "paid" } },
  { $sort: { created_at: -1 } },
  { $limit: 10 }
])
```

因为先 `$project` 可能让优化和索引利用变复杂。通常先过滤，再排序/限制，再做最终形状转换。

## 8. $count

统计已支付订单数量：

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  { $count: "paid_order_count" }
])
```

输出：

```javascript
{ paid_order_count: 4 }
```

## 9. 本节练习

完成下面聚合：

1. 查询已支付订单，只输出 `order_no`、`total_amount`、`paid_at`。
2. 查询金额最高的 2 个订单。
3. 查询 2026-07-02 之后创建的订单。
4. 查询上架商品，只输出商品名、价格、分类。
5. 把商品价格从分转成元输出。
6. 查询发布文章，按浏览量倒序。
7. 统计可见评论数量。
8. 对订单做第二页分页，每页 2 条。

## 10. 本节小结

你需要记住：

- `$match` 过滤文档，通常尽量前置。
- `$project` 控制输出字段，也能计算字段。
- `$sort` 排序。
- `$limit` 限制数量。
- `$skip` 跳过数量，但深分页要谨慎。
- `$count` 输出统计数量。

