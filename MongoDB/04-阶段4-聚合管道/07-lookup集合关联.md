# 07. lookup 集合关联

`$lookup` 用于在聚合管道中关联另一个集合。它很有用，但不是万能 join，更不是所有查询的默认答案。

## 1. 基本用法

订单关联用户：

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  {
    $lookup: {
      from: "users",
      localField: "user_id",
      foreignField: "_id",
      as: "user"
    }
  }
])
```

含义：

- `from`：要关联的集合。
- `localField`：当前集合字段。
- `foreignField`：被关联集合字段。
- `as`：关联结果放到哪个字段。

注意：`user` 是数组，即使只匹配到一个用户。

## 2. lookup 后 unwind

如果每个订单只对应一个用户，可以展开：

```javascript
db.orders.aggregate([
  { $match: { status: "paid" } },
  {
    $lookup: {
      from: "users",
      localField: "user_id",
      foreignField: "_id",
      as: "user"
    }
  },
  { $unwind: "$user" },
  {
    $project: {
      _id: 0,
      order_no: 1,
      total_amount: 1,
      user_name: "$user.name",
      user_email: "$user.email"
    }
  }
])
```

## 3. 用户消费排行榜带用户名

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
  { $limit: 10 },
  {
    $lookup: {
      from: "users",
      localField: "_id",
      foreignField: "_id",
      as: "user"
    }
  },
  { $unwind: "$user" },
  {
    $project: {
      _id: 0,
      user_id: "$_id",
      user_name: "$user.name",
      city: "$user.city",
      order_count: 1,
      total_amount: 1
    }
  }
])
```

注意这里先 `$group`、`$sort`、`$limit`，再 `$lookup`。这样只关联 Top 10 用户，而不是所有订单都关联用户。

## 4. 文章评论数关联

查询文章并关联评论：

```javascript
db.posts.aggregate([
  { $match: { status: "published" } },
  {
    $lookup: {
      from: "comments",
      localField: "_id",
      foreignField: "post_id",
      as: "comments"
    }
  },
  {
    $project: {
      _id: 0,
      title: 1,
      views: "$stats.views",
      likes: "$stats.likes",
      comment_count: { $size: "$comments" }
    }
  }
])
```

这适合小数据量练习，但大数据下不建议为了统计评论数把所有 comments 拉进数组。更好的方式通常是：

- 文章文档冗余 `stats.comments`。
- 或从 comments 集合 `$group` 后再关联。

## 5. lookup pipeline 写法

更灵活的 `$lookup` 可以使用 pipeline。

查询文章，并只关联可见评论：

```javascript
db.posts.aggregate([
  { $match: { status: "published" } },
  {
    $lookup: {
      from: "comments",
      let: { postId: "$_id" },
      pipeline: [
        {
          $match: {
            $expr: {
              $and: [
                { $eq: ["$post_id", "$$postId"] },
                { $eq: ["$status", "visible"] }
              ]
            }
          }
        },
        { $sort: { created_at: -1 } },
        { $limit: 3 },
        {
          $project: {
            _id: 1,
            content: 1,
            created_at: 1
          }
        }
      ],
      as: "recent_comments"
    }
  }
])
```

这种写法适合关联时需要过滤、排序、限制、投影。

## 6. lookup 的边界

`$lookup` 不适合无脑用在：

- 高频接口的大集合关联。
- 没有前置 `$match` 的全量关联。
- 关联结果数组可能很大的场景。
- 能通过建模冗余解决的列表展示。

例如文章列表展示作者昵称和头像，很多时候更适合在 `posts.author_snapshot` 中冗余，而不是每次 `$lookup users`。

## 7. lookup 和索引

被关联集合的 `foreignField` 应该有索引。

例如：

```javascript
db.comments.createIndex({ post_id: 1, created_at: -1 })
```

订单关联用户时，`users._id` 默认有索引。

如果 `$lookup` pipeline 中用到：

```javascript
{ $eq: ["$post_id", "$$postId"] }
```

也要考虑 `comments.post_id` 索引。

## 8. 关联缺失数据

如果 `$lookup` 没有关联到数据，结果数组为空。

如果后面直接：

```javascript
{ $unwind: "$user" }
```

该文档会被丢弃。

如果想保留：

```javascript
{
  $unwind: {
    path: "$user",
    preserveNullAndEmptyArrays: true
  }
}
```

## 9. 本节练习

完成：

1. 查询已支付订单，并关联用户名称。
2. 查询用户消费排行榜，输出用户名称和消费总额。
3. 查询发布文章，并关联作者名称。
4. 查询发布文章，并关联最近 3 条可见评论。
5. 查询每篇文章的可见评论数量。
6. 尝试使用 `preserveNullAndEmptyArrays` 保留关联不到用户的订单。
7. 写出你认为哪些场景不应该用 `$lookup`。

## 10. 本节小结

你需要记住：

- `$lookup` 关联结果是数组。
- 一对一关联后常接 `$unwind`。
- 先过滤、分组、限制，再 lookup，通常更高效。
- `$lookup` pipeline 写法更灵活。
- `$lookup` 不是万能 join，高频列表更要考虑建模和冗余。
- 被关联字段要考虑索引。

