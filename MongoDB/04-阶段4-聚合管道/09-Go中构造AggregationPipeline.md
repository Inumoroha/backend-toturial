# 09. Go 中构造 Aggregation Pipeline

本节把聚合管道搬到 Go Driver v2 中执行。核心方法是 `collection.Aggregate(ctx, pipeline)`，返回 Cursor。

## 1. Go 中的 pipeline 类型

Go Driver v2 中常用：

```go
pipeline := mongo.Pipeline{
    bson.D{{"$match", bson.D{{"status", "paid"}}}},
    bson.D{{"$group", bson.D{
        {"_id", "$user_id"},
        {"order_count", bson.D{{"$sum", 1}}},
        {"total_amount", bson.D{{"$sum", "$total_amount"}}},
    }}},
    bson.D{{"$sort", bson.D{{"total_amount", -1}}}},
}
```

需要导入：

```go
import (
    "go.mongodb.org/mongo-driver/v2/bson"
    "go.mongodb.org/mongo-driver/v2/mongo"
)
```

`mongo.Pipeline` 本质上是 `[]bson.D`，适合表达有序 stage。

## 2. 执行 Aggregate

```go
cursor, err := collection.Aggregate(ctx, pipeline)
if err != nil {
    return nil, err
}
defer cursor.Close(ctx)
```

和 `Find` 一样，聚合也返回 Cursor。

你需要：

- Close。
- Decode。
- 检查 cursor.Err。

## 3. 定义结果结构体

用户消费排行结果：

```go
type UserSpending struct {
    UserID      bson.ObjectID `bson:"user_id" json:"user_id"`
    OrderCount  int64         `bson:"order_count" json:"order_count"`
    TotalAmount int64         `bson:"total_amount" json:"total_amount"`
}
```

注意：如果聚合输出 `_id`，你可以：

1. 结构体里定义 `ID bson.ObjectID `bson:"_id"``
2. 或在 pipeline 里 `$project` 改成 `user_id`

推荐输出 API 友好的字段：

```javascript
{ $project: { _id: 0, user_id: "$_id", order_count: 1, total_amount: 1 } }
```

## 4. Go 示例：用户消费排行

```go
func UserSpendingRanking(ctx context.Context, orders *mongo.Collection, limit int64) ([]UserSpending, error) {
    if limit <= 0 || limit > 100 {
        limit = 10
    }

    pipeline := mongo.Pipeline{
        bson.D{{"$match", bson.D{{"status", "paid"}}}},
        bson.D{{"$group", bson.D{
            {"_id", "$user_id"},
            {"order_count", bson.D{{"$sum", 1}}},
            {"total_amount", bson.D{{"$sum", "$total_amount"}}},
        }}},
        bson.D{{"$sort", bson.D{{"total_amount", -1}}}},
        bson.D{{"$limit", limit}},
        bson.D{{"$project", bson.D{
            {"_id", 0},
            {"user_id", "$_id"},
            {"order_count", 1},
            {"total_amount", 1},
        }}},
    }

    cursor, err := orders.Aggregate(ctx, pipeline)
    if err != nil {
        return nil, err
    }
    defer cursor.Close(ctx)

    var results []UserSpending
    if err := cursor.All(ctx, &results); err != nil {
        return nil, err
    }

    return results, nil
}
```

这里结果有限，所以使用 `cursor.All` 可以接受。

## 5. Go 示例：按天统计订单金额

结果结构：

```go
type DailyOrderAmount struct {
    Day         string `bson:"day" json:"day"`
    OrderCount  int64  `bson:"order_count" json:"order_count"`
    TotalAmount int64  `bson:"total_amount" json:"total_amount"`
}
```

函数：

```go
func DailyPaidOrderAmount(ctx context.Context, orders *mongo.Collection, start, end time.Time) ([]DailyOrderAmount, error) {
    pipeline := mongo.Pipeline{
        bson.D{{"$match", bson.D{
            {"status", "paid"},
            {"paid_at", bson.D{
                {"$gte", start},
                {"$lt", end},
            }},
        }}},
        bson.D{{"$group", bson.D{
            {"_id", bson.D{{"$dateToString", bson.D{
                {"format", "%Y-%m-%d"},
                {"date", "$paid_at"},
                {"timezone", "Asia/Shanghai"},
            }}}},
            {"order_count", bson.D{{"$sum", 1}}},
            {"total_amount", bson.D{{"$sum", "$total_amount"}}},
        }}},
        bson.D{{"$project", bson.D{
            {"_id", 0},
            {"day", "$_id"},
            {"order_count", 1},
            {"total_amount", 1},
        }}},
        bson.D{{"$sort", bson.D{{"day", 1}}}},
    }

    cursor, err := orders.Aggregate(ctx, pipeline)
    if err != nil {
        return nil, err
    }
    defer cursor.Close(ctx)

    var results []DailyOrderAmount
    if err := cursor.All(ctx, &results); err != nil {
        return nil, err
    }

    return results, nil
}
```

需要导入：

```go
import "time"
```

## 6. AggregateOptions

可以设置选项：

```go
opts := options.Aggregate().SetAllowDiskUse(true)

cursor, err := collection.Aggregate(ctx, pipeline, opts)
```

需要导入：

```go
"go.mongodb.org/mongo-driver/v2/mongo/options"
```

`AllowDiskUse` 允许部分聚合操作使用磁盘临时文件。学习阶段先了解，真实项目中要结合性能和资源评估。

## 7. 动态构造 pipeline

根据状态过滤：

```go
match := bson.D{}

if status != "" {
    match = append(match, bson.E{Key: "status", Value: status})
}

pipeline := mongo.Pipeline{}

if len(match) > 0 {
    pipeline = append(pipeline, bson.D{{"$match", match}})
}

pipeline = append(pipeline,
    bson.D{{"$group", bson.D{
        {"_id", "$user_id"},
        {"count", bson.D{{"$sum", 1}}},
    }}},
)
```

动态 pipeline 要注意：

- stage 顺序仍然重要。
- 参数要校验。
- 不要把用户输入直接拼成任意字段名。

## 8. Go 中常见错误

### 用 bson.M 表达排序

不推荐：

```go
bson.M{"total_amount": -1, "order_count": -1}
```

推荐：

```go
bson.D{{"total_amount", -1}, {"order_count", -1}}
```

排序字段顺序有意义，用 `bson.D`。

### 忘记 Close Cursor

```go
cursor, err := collection.Aggregate(ctx, pipeline)
defer cursor.Close(ctx)
```

### Decode 字段不匹配

如果 pipeline 输出：

```javascript
{ _id: "...", total_amount: 100 }
```

但结构体写：

```go
type Result struct {
    UserID string `bson:"user_id"`
}
```

就 decode 不到 `user_id`。要么改 `$project`，要么改结构体 tag。

## 9. 本节练习

用 Go 实现：

1. 用户消费排行榜。
2. 按天统计订单金额。
3. 商品销量排行榜。
4. 按订单状态统计数量。
5. 发布文章浏览量排行。

每个函数要求：

- 接收 `context.Context`。
- 使用 `mongo.Pipeline`。
- 正确 Close Cursor。
- 定义结果结构体。
- 返回结构体 slice。

## 10. 本节小结

你需要记住：

- Go 中使用 `collection.Aggregate(ctx, pipeline)`。
- 推荐使用 `mongo.Pipeline` 和 `bson.D`。
- 聚合返回 Cursor，需要关闭。
- 输出结构最好用 `$project` 整理成易 Decode 的字段。
- 动态 pipeline 要注意 stage 顺序和参数校验。

