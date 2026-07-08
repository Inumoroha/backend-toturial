# 06. 多文档查询、Cursor、排序与分页

本节学习 `Find`。和 `FindOne` 不同，`Find` 返回 Cursor，需要你正确关闭、遍历和检查错误。

## 1. ListFilter

在 `internal/user/model.go` 或单独文件中定义：

```go
type ListFilter struct {
    Status string
    Role   string
    Limit  int64
    Offset int64
}
```

学习阶段使用 `Offset`，后面性能阶段会学习游标分页。

## 2. 构造 filter

```go
func buildListFilter(input ListFilter) bson.D {
    filter := bson.D{
        {"deleted_at", bson.D{{"$exists", false}}},
    }

    if input.Status != "" {
        filter = append(filter, bson.E{Key: "status", Value: input.Status})
    }

    if input.Role != "" {
        filter = append(filter, bson.E{Key: "roles", Value: input.Role})
    }

    return filter
}
```

注意：

- `roles` 是数组字段，`{"roles", "admin"}` 可以匹配数组包含 admin 的文档。
- 条件是动态的，所以用 append 构造。

## 3. List 方法

```go
func (r *Repository) List(ctx context.Context, input ListFilter) ([]User, error) {
    filter := buildListFilter(input)

    limit := input.Limit
    if limit <= 0 || limit > 100 {
        limit = 20
    }

    offset := input.Offset
    if offset < 0 {
        offset = 0
    }

    opts := options.Find().
        SetSort(bson.D{{"created_at", -1}}).
        SetSkip(offset).
        SetLimit(limit)

    cursor, err := r.collection.Find(ctx, filter, opts)
    if err != nil {
        return nil, err
    }
    defer cursor.Close(ctx)

    users := make([]User, 0)
    for cursor.Next(ctx) {
        var u User
        if err := cursor.Decode(&u); err != nil {
            return nil, err
        }
        users = append(users, u)
    }

    if err := cursor.Err(); err != nil {
        return nil, err
    }

    return users, nil
}
```

需要导入：

```go
import (
    "context"

    "go.mongodb.org/mongo-driver/v2/bson"
    "go.mongodb.org/mongo-driver/v2/mongo/options"
)
```

## 4. Cursor 的正确处理

`Find` 返回：

```go
cursor, err := collection.Find(ctx, filter, opts)
```

必须处理：

```go
defer cursor.Close(ctx)
```

遍历：

```go
for cursor.Next(ctx) {
    var u User
    if err := cursor.Decode(&u); err != nil {
        return nil, err
    }
}
```

最后检查：

```go
if err := cursor.Err(); err != nil {
    return nil, err
}
```

这三步不要漏。

## 5. cursor.All

如果数据量确定不大，可以使用：

```go
func (r *Repository) ListWithAll(ctx context.Context, input ListFilter) ([]User, error) {
    filter := buildListFilter(input)

    opts := options.Find().
        SetSort(bson.D{{"created_at", -1}}).
        SetLimit(20)

    cursor, err := r.collection.Find(ctx, filter, opts)
    if err != nil {
        return nil, err
    }
    defer cursor.Close(ctx)

    var users []User
    if err := cursor.All(ctx, &users); err != nil {
        return nil, err
    }

    return users, nil
}
```

`All` 会把结果一次性读入内存。列表有明确 limit 时可以用；没有 limit 的大查询不要用。

## 6. 排序

按创建时间倒序：

```go
opts := options.Find().SetSort(bson.D{{"created_at", -1}})
```

多字段排序：

```go
opts := options.Find().SetSort(bson.D{
    {"status", 1},
    {"created_at", -1},
})
```

排序用 `bson.D`，不要用 `bson.M`。因为排序字段顺序有意义。

## 7. 分页

简单分页：

```go
opts := options.Find().
    SetSkip((page - 1) * pageSize).
    SetLimit(pageSize)
```

在 Repository 中可以接收：

```go
type ListFilter struct {
    Limit  int64
    Offset int64
}
```

API 层可以把 `page/page_size` 转成 `offset/limit`。

注意：`skip` 大了会慢。后面索引和性能阶段会改为游标分页。

## 8. 统计数量

```go
func (r *Repository) Count(ctx context.Context, input ListFilter) (int64, error) {
    filter := buildListFilter(input)
    return r.collection.CountDocuments(ctx, filter)
}
```

列表接口是否返回 total，要看业务需要。大集合频繁 count 也有成本。

## 9. 投影

如果列表不需要返回全部字段：

```go
opts := options.Find().
    SetProjection(bson.D{
        {"email", 1},
        {"name", 1},
        {"status", 1},
        {"created_at", 1},
    }).
    SetSort(bson.D{{"created_at", -1}}).
    SetLimit(limit)
```

投影能减少网络传输和解码成本，但不要过早复杂化。先明确接口需要什么字段。

## 10. 本节常见错误

### 忘记 Close Cursor

```go
cursor, err := r.collection.Find(ctx, filter)
// 忘记 defer cursor.Close(ctx)
```

这会造成资源泄漏。

### 忘记检查 cursor.Err

遍历过程中可能发生错误，最后要检查：

```go
if err := cursor.Err(); err != nil {
    return nil, err
}
```

### 无限制查询

危险：

```go
cursor, err := r.collection.Find(ctx, bson.D{})
```

如果集合很大，这会造成内存和网络压力。

列表查询应该有合理 limit。

## 11. 本节验收

你应该能做到：

- 使用 `Find` 查询多条文档。
- 正确 `defer cursor.Close(ctx)`。
- 使用 `cursor.Next` 和 `cursor.Decode`。
- 使用 `cursor.All` 并说出适用场景。
- 使用 `options.Find().SetSort().SetSkip().SetLimit()`。
- 解释为什么深分页会有性能问题。

