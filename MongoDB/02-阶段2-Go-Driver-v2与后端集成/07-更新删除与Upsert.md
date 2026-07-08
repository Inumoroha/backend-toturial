# 07. 更新、删除与 Upsert

本节用 Go Driver 实现更新、软删除和 upsert。重点是安全构造 update 文档，避免误更新和零值覆盖。

## 1. UpdateProfileInput

```go
type UpdateProfileInput struct {
    Name *string
    Age  *int
}
```

为什么用指针？

- `nil` 表示用户没有传这个字段。
- 非 nil 表示用户明确要更新。
- 可以区分 `age` 未传和 `age = 0`。

## 2. UpdateProfile

```go
func (r *Repository) UpdateProfile(ctx context.Context, id bson.ObjectID, input UpdateProfileInput) error {
    set := bson.D{}

    if input.Name != nil {
        set = append(set, bson.E{Key: "name", Value: *input.Name})
    }

    if input.Age != nil {
        set = append(set, bson.E{Key: "age", Value: *input.Age})
    }

    if len(set) == 0 {
        return ErrInvalidUpdate
    }

    set = append(set, bson.E{Key: "updated_at", Value: time.Now().UTC()})

    filter := bson.D{
        {"_id", id},
        {"deleted_at", bson.D{{"$exists", false}}},
    }

    update := bson.D{
        {"$set", set},
    }

    result, err := r.collection.UpdateOne(ctx, filter, update)
    if err != nil {
        return err
    }

    if result.MatchedCount == 0 {
        return ErrNotFound
    }

    return nil
}
```

需要导入：

```go
import (
    "context"
    "time"

    "go.mongodb.org/mongo-driver/v2/bson"
)
```

## 3. UpdateOne 返回值

`UpdateOne` 返回 `UpdateResult`。

常用字段：

- `MatchedCount`：匹配到几条。
- `ModifiedCount`：实际修改了几条。
- `UpsertedCount`：upsert 插入了几条。
- `UpsertedID`：upsert 插入文档的 ID。

注意：

```text
MatchedCount = 1, ModifiedCount = 0
```

不一定是错误，可能是新值和旧值相同。

## 4. 增加角色

```go
func (r *Repository) AddRole(ctx context.Context, id bson.ObjectID, role string) error {
    filter := bson.D{
        {"_id", id},
        {"deleted_at", bson.D{{"$exists", false}}},
    }

    update := bson.D{
        {"$addToSet", bson.D{{"roles", role}}},
        {"$set", bson.D{{"updated_at", time.Now().UTC()}}},
    }

    result, err := r.collection.UpdateOne(ctx, filter, update)
    if err != nil {
        return err
    }
    if result.MatchedCount == 0 {
        return ErrNotFound
    }

    return nil
}
```

角色不希望重复，所以用 `$addToSet`，不用 `$push`。

## 5. 移除角色

```go
func (r *Repository) RemoveRole(ctx context.Context, id bson.ObjectID, role string) error {
    filter := bson.D{
        {"_id", id},
        {"deleted_at", bson.D{{"$exists", false}}},
    }

    update := bson.D{
        {"$pull", bson.D{{"roles", role}}},
        {"$set", bson.D{{"updated_at", time.Now().UTC()}}},
    }

    result, err := r.collection.UpdateOne(ctx, filter, update)
    if err != nil {
        return err
    }
    if result.MatchedCount == 0 {
        return ErrNotFound
    }

    return nil
}
```

## 6. 软删除

```go
func (r *Repository) SoftDelete(ctx context.Context, id bson.ObjectID) error {
    now := time.Now().UTC()

    filter := bson.D{
        {"_id", id},
        {"deleted_at", bson.D{{"$exists", false}}},
    }

    update := bson.D{
        {"$set", bson.D{
            {"deleted_at", now},
            {"updated_at", now},
        }},
    }

    result, err := r.collection.UpdateOne(ctx, filter, update)
    if err != nil {
        return err
    }
    if result.MatchedCount == 0 {
        return ErrNotFound
    }

    return nil
}
```

软删除后，`FindByID`、`FindByEmail`、`List` 都应该过滤：

```go
{"deleted_at", bson.D{{"$exists", false}}}
```

## 7. 物理删除

```go
func (r *Repository) DeleteHard(ctx context.Context, id bson.ObjectID) error {
    result, err := r.collection.DeleteOne(ctx, bson.D{{"_id", id}})
    if err != nil {
        return err
    }
    if result.DeletedCount == 0 {
        return ErrNotFound
    }

    return nil
}
```

真实业务中，用户、订单、支付这类数据通常不建议随意物理删除。物理删除更适合测试数据、临时数据或明确可丢弃的数据。

## 8. Upsert

Upsert：有则更新，没有则插入。

```go
func (r *Repository) UpsertByEmail(ctx context.Context, u *User) error {
    now := time.Now().UTC()

    if u.ID.IsZero() {
        u.ID = bson.NewObjectID()
    }
    if u.Status == "" {
        u.Status = "active"
    }

    filter := bson.D{{"email", u.Email}}

    update := bson.D{
        {"$set", bson.D{
            {"name", u.Name},
            {"age", u.Age},
            {"roles", u.Roles},
            {"status", u.Status},
            {"updated_at", now},
        }},
        {"$setOnInsert", bson.D{
            {"_id", u.ID},
            {"email", u.Email},
            {"created_at", now},
        }},
    }

    opts := options.UpdateOne().SetUpsert(true)

    result, err := r.collection.UpdateOne(ctx, filter, update, opts)
    if err != nil {
        return err
    }

    if result.UpsertedID != nil {
        if id, ok := result.UpsertedID.(bson.ObjectID); ok {
            u.ID = id
        }
    }

    return nil
}
```

需要导入：

```go
"go.mongodb.org/mongo-driver/v2/mongo/options"
```

## 9. update 文档必须使用更新操作符

正确：

```go
update := bson.D{
    {"$set", bson.D{{"name", "Alice New"}}},
}
```

错误：

```go
update := bson.D{
    {"name", "Alice New"},
}
```

`UpdateOne` 的 update 参数应该是包含更新操作符的文档，例如 `$set`、`$inc`、`$addToSet`。

如果你要替换整条文档，使用 `ReplaceOne`，但要谨慎。

## 10. 本节验收

你应该能做到：

- 使用 `UpdateOne` 更新指定用户。
- 使用指针输入避免零值覆盖。
- 使用 `$addToSet` 添加角色。
- 使用 `$pull` 移除角色。
- 使用软删除代替物理删除。
- 使用 `DeleteOne` 并检查 `DeletedCount`。
- 使用 `options.UpdateOne().SetUpsert(true)`。
- 解释 `MatchedCount` 和 `ModifiedCount` 的区别。

