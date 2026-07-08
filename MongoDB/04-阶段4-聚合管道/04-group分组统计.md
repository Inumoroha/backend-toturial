# 04. group 分组统计

`$group` 是聚合管道中最重要的 stage 之一。它可以按字段分组，然后做求和、计数、平均值、最大值、最小值等统计。

## 1. $group 基本结构

```javascript
{
  $group: {
    _id: "$分组字段",
    字段名: { 聚合操作: "$被统计字段" }
  }
}
```

例如按订单状态统计数量：

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$status",
      count: { $sum: 1 }
    }
  }
])
```

输出类似：

```javascript
{ _id: "paid", count: 4 }
{ _id: "pending_payment", count: 1 }
{ _id: "cancelled", count: 1 }
```

## 2. _id 是分组键

`$group` 里的 `_id` 不是原文档 `_id`，而是分组键。

按用户分组：

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  {
    $group: {
      _id: "$user_id",
      order_count: { $sum: 1 },
      total_amount: { $sum: "$total_amount" }
    }
  }
])
```

这里输出里的 `_id` 是 `user_id`。

## 3. 统计总金额

统计全部已支付订单金额：

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  {
    $group: {
      _id: null,
      total_amount: { $sum: "$total_amount" },
      order_count: { $sum: 1 }
    }
  }
])
```

`_id: null` 表示不分组，把所有输入文档归为一组。

## 4. 平均值、最大值、最小值

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  {
    $group: {
      _id: null,
      avg_amount: { $avg: "$total_amount" },
      max_amount: { $max: "$total_amount" },
      min_amount: { $min: "$total_amount" }
    }
  }
])
```

常见 accumulator：

| 操作 | 作用 |
| --- | --- |
| `$sum` | 求和或计数 |
| `$avg` | 平均值 |
| `$max` | 最大值 |
| `$min` | 最小值 |
| `$first` | 当前排序下第一条 |
| `$last` | 当前排序下最后一条 |
| `$push` | 收集为数组 |
| `$addToSet` | 去重收集为数组 |

## 5. 按多个字段分组

按状态和用户分组：

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: {
        status: "$status",
        user_id: "$user_id"
      },
      count: { $sum: 1 },
      total_amount: { $sum: "$total_amount" }
    }
  }
])
```

输出 `_id` 是一个对象：

```javascript
{
  _id: {
    status: "paid",
    user_id: ObjectId("...")
  },
  count: 2,
  total_amount: 55500
}
```

## 6. 分组后整理输出

用 `$project` 改造输出结构：

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  {
    $group: {
      _id: "$user_id",
      order_count: { $sum: 1 },
      total_amount: { $sum: "$total_amount" }
    }
  },
  {
    $project: {
      _id: 0,
      user_id: "$_id",
      order_count: 1,
      total_amount: 1,
      total_yuan: { $divide: ["$total_amount", 100] }
    }
  },
  { $sort: { total_amount: -1 } }
])
```

这样输出更适合 API 返回。

## 7. 用户消费排行榜

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  {
    $group: {
      _id: "$user_id",
      order_count: { $sum: 1 },
      total_amount: { $sum: "$total_amount" }
    }
  },
  { $sort: { total_amount: -1 } },
  { $limit: 10 }
])
```

这就是用户消费 Top 10。

后面 `$lookup` 会把用户名称关联出来。

## 8. 按订单状态统计

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$status",
      count: { $sum: 1 },
      total_amount: { $sum: "$total_amount" }
    }
  },
  { $sort: { count: -1 } }
])
```

注意：取消订单金额是否要算入 total，要看业务。很多报表只统计 paid。

## 9. $push 和 $addToSet

收集每个用户的订单号：

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$user_id",
      order_nos: { $push: "$order_no" }
    }
  }
])
```

收集每个用户买过的订单状态并去重：

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$user_id",
      statuses: { $addToSet: "$status" }
    }
  }
])
```

注意：不要在大数据量下无限 `$push` 巨大数组，可能造成内存压力。

## 10. 本节练习

完成下面聚合：

1. 按订单状态统计订单数。
2. 统计已支付订单总金额和订单数。
3. 按用户统计消费总额。
4. 找出消费最高的 3 个用户。
5. 按用户统计订单平均金额。
6. 按商品分类统计商品数量。
7. 统计发布文章总浏览量和总点赞数。
8. 按文章状态统计文章数量。

## 11. 本节小结

你需要记住：

- `$group` 的 `_id` 是分组键。
- `$sum: 1` 常用于计数。
- `$sum: "$field"` 用于对字段求和。
- `_id: null` 表示所有文档归为一组。
- 分组后常用 `$project` 调整输出结构。
- 大规模 `$push` 要谨慎。

