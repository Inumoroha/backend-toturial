# 05. addFields / set 字段计算

`$addFields` 用于给文档添加新字段，也可以覆盖已有字段。`$set` 是 `$addFields` 的别名，在聚合管道中二者常用于字段计算和格式整理。

## 1. 基本用法

给订单增加金额元字段：

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  {
    $addFields: {
      total_yuan: { $divide: ["$total_amount", 100] }
    }
  }
])
```

输出中会保留原字段，并多出 `total_yuan`。

## 2. $set 等价写法

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  {
    $set: {
      total_yuan: { $divide: ["$total_amount", 100] }
    }
  }
])
```

在聚合管道中，`$set` 可读性更像“设置字段”，本教程后面两种都会见到。

## 3. 计算订单商品数量

`$size` 可以计算数组长度：

```javascript
db.orders.aggregate([
  {
    $addFields: {
      item_count: { $size: "$items" }
    }
  },
  {
    $project: {
      _id: 0,
      order_no: 1,
      item_count: 1
    }
  }
])
```

注意：`$size` 要求字段是数组。如果字段可能缺失，需要用 `$ifNull` 兜底。

```javascript
item_count: { $size: { $ifNull: ["$items", []] } }
```

## 4. 条件字段 $cond

根据订单金额标记大额订单：

```javascript
db.orders.aggregate([
  {
    $addFields: {
      amount_level: {
        $cond: [
          { $gte: ["$total_amount", 20000] },
          "high",
          "normal"
        ]
      }
    }
  },
  {
    $project: {
      _id: 0,
      order_no: 1,
      total_amount: 1,
      amount_level: 1
    }
  }
])
```

## 5. 多分支 $switch

```javascript
db.orders.aggregate([
  {
    $addFields: {
      amount_level: {
        $switch: {
          branches: [
            { case: { $gte: ["$total_amount", 30000] }, then: "high" },
            { case: { $gte: ["$total_amount", 10000] }, then: "middle" }
          ],
          default: "low"
        }
      }
    }
  },
  {
    $project: {
      _id: 0,
      order_no: 1,
      total_amount: 1,
      amount_level: 1
    }
  }
])
```

## 6. 字符串拼接

用户标签：

```javascript
db.users.aggregate([
  {
    $addFields: {
      label: { $concat: ["$name", " <", "$email", ">"] }
    }
  },
  {
    $project: {
      _id: 0,
      label: 1,
      city: 1
    }
  }
])
```

## 7. 处理空值 $ifNull

有些订单没有 `paid_at`：

```javascript
db.orders.aggregate([
  {
    $addFields: {
      paid_at_display: {
        $ifNull: ["$paid_at", "not paid"]
      }
    }
  },
  {
    $project: {
      _id: 0,
      order_no: 1,
      status: 1,
      paid_at_display: 1
    }
  }
])
```

## 8. 覆盖已有字段

可以覆盖已有字段，但要谨慎。

例如把 `total_amount` 临时转为元：

```javascript
db.orders.aggregate([
  {
    $set: {
      total_amount: { $divide: ["$total_amount", 100] }
    }
  }
])
```

这只影响聚合输出，不会修改数据库原文档。

如果你想真正修改集合，要用 update，不是 aggregate。

## 9. $addFields 和 $project 的区别

`$addFields`：

- 保留所有原字段。
- 添加或覆盖指定字段。

`$project`：

- 控制最终输出哪些字段。
- 可以排除不需要的字段。
- 也可以计算字段。

常见组合：

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  { $addFields: { total_yuan: { $divide: ["$total_amount", 100] } } },
  { $project: { _id: 0, order_no: 1, total_yuan: 1 } }
])
```

## 10. 本节练习

完成：

1. 给订单增加 `total_yuan` 字段。
2. 给订单增加 `item_count` 字段。
3. 按金额给订单分成 `high`、`middle`、`low`。
4. 给用户增加 `label = name <email>`。
5. 给未支付订单显示 `paid_at_display = "not paid"`。
6. 给商品增加 `price_yuan` 字段。
7. 给文章增加 `hot` 字段，浏览量大于 400 为 true。

## 11. 本节小结

你需要记住：

- `$addFields` 添加或覆盖字段，并保留原字段。
- `$set` 是 `$addFields` 的别名。
- `$project` 更适合整理最终输出。
- `$cond` 处理简单条件。
- `$switch` 处理多分支。
- `$ifNull` 处理缺失或空值。
- 聚合中的字段覆盖不修改原始集合。

