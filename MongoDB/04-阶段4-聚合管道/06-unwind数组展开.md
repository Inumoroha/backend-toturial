# 06. unwind 数组展开

`$unwind` 用于把数组字段拆成多条文档。它在订单商品统计、标签统计、评论统计中非常常用。

## 1. 为什么需要 $unwind

订单文档中 `items` 是数组：

```javascript
{
  order_no: "O202607010001",
  items: [
    { product_name: "MongoDB 入门课", quantity: 1 },
    { product_name: "Docker 工具包", quantity: 1 }
  ]
}
```

如果要统计商品销量，需要把每个 item 当作一条统计记录。

`$unwind` 就是做这个。

## 2. 基本用法

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  { $unwind: "$items" },
  {
    $project: {
      _id: 0,
      order_no: 1,
      product_name: "$items.product_name",
      quantity: "$items.quantity"
    }
  }
])
```

原来一条包含 2 个 items 的订单，会变成 2 条输出。

## 3. 商品销量排行榜

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  { $unwind: "$items" },
  {
    $group: {
      _id: "$items.product_id",
      product_name: { $first: "$items.product_name" },
      sales_quantity: { $sum: "$items.quantity" },
      sales_amount: {
        $sum: { $multiply: ["$items.price", "$items.quantity"] }
      }
    }
  },
  { $sort: { sales_quantity: -1 } }
])
```

要点：

- 先 `$match` 只统计已支付订单。
- `$unwind` 展开商品项。
- `$group` 按商品分组。
- `$sum` 统计销量和销售额。

## 4. 统计标签出现次数

文章 tags：

```javascript
db.posts.aggregate([
  { $match: { status: "published" } },
  { $unwind: "$tags" },
  {
    $group: {
      _id: "$tags",
      count: { $sum: 1 }
    }
  },
  { $sort: { count: -1 } }
])
```

输出：

```javascript
{ _id: "mongodb", count: 2 }
{ _id: "aggregation", count: 1 }
{ _id: "go", count: 1 }
```

## 5. preserveNullAndEmptyArrays

默认情况下，如果数组为空或字段不存在，`$unwind` 会丢弃该文档。

如果想保留：

```javascript
db.orders.aggregate([
  {
    $unwind: {
      path: "$items",
      preserveNullAndEmptyArrays: true
    }
  }
])
```

适合需要保留没有数组元素的主文档的场景。

## 6. includeArrayIndex

保留数组下标：

```javascript
db.orders.aggregate([
  {
    $unwind: {
      path: "$items",
      includeArrayIndex: "item_index"
    }
  },
  {
    $project: {
      _id: 0,
      order_no: 1,
      item_index: 1,
      product_name: "$items.product_name"
    }
  }
])
```

输出会包含 `item_index`。

## 7. $unwind 会放大数据量

如果：

```text
10000 条订单
平均每条 5 个 items
```

`$unwind` 后就是：

```text
50000 条中间文档
```

所以 `$unwind` 前尽量 `$match`，只处理必要数据。

推荐：

```javascript
[
  { $match: { status: "paid" } },
  { $unwind: "$items" },
  { $group: { ... } }
]
```

不推荐：

```javascript
[
  { $unwind: "$items" },
  { $match: { status: "paid" } },
  { $group: { ... } }
]
```

## 8. 数组内字段过滤

统计 MongoDB 入门课销量：

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  { $unwind: "$items" },
  {
    $match: {
      "items.product_id": ObjectId("661000000000000000000001")
    }
  },
  {
    $group: {
      _id: "$items.product_id",
      product_name: { $first: "$items.product_name" },
      quantity: { $sum: "$items.quantity" }
    }
  }
])
```

这里第二个 `$match` 是在展开后过滤具体 item。

## 9. 本节练习

完成：

1. 展开订单 items，输出订单号、商品名、数量。
2. 统计每个商品销量。
3. 统计每个商品销售额。
4. 找出销量最高的商品。
5. 展开文章 tags，统计标签热度。
6. 展开订单 items 后，只统计 Go 后端实战销量。
7. 使用 `includeArrayIndex` 输出订单商品下标。

## 10. 本节小结

你需要记住：

- `$unwind` 会把数组拆成多条文档。
- `$unwind` 常用于商品销量、标签统计。
- `$unwind` 会放大数据量，要先 `$match`。
- `preserveNullAndEmptyArrays` 可以保留空数组文档。
- `includeArrayIndex` 可以输出数组下标。

