# 03. BSON 类型与 Struct 标签

MongoDB 存储 BSON，Go 程序使用结构体、map 或 `bson.D` 构造数据。理解 BSON 映射，是写对 Repository 的基础。

## 1. v2 常用 BSON 包

Go Driver v2 中常用导入：

```go
import "go.mongodb.org/mongo-driver/v2/bson"
```

常用类型：

| 类型 | 说明 | 常用场景 |
| --- | --- | --- |
| `bson.D` | 有序文档 | 查询条件、排序、命令 |
| `bson.M` | 无序 map | 简单过滤、临时文档 |
| `bson.A` | 数组 | `$in`、聚合数组 |
| `bson.E` | `bson.D` 中的单个元素 | 较少单独使用 |
| `bson.ObjectID` | BSON ObjectId | `_id` 字段 |

官方 BSON 文档也强调：`D` 是有序文档，`M` 是无序文档，`A` 是数组。

## 2. bson.D

`bson.D` 是有序的：

```go
filter := bson.D{
    {"status", "active"},
    {"age", bson.D{{"$gte", 18}}},
}
```

适合：

- 查询过滤条件。
- 排序。
- MongoDB 命令。
- 需要字段顺序的场景。

排序必须保留顺序：

```go
sort := bson.D{
    {"status", 1},
    {"created_at", -1},
}
```

## 3. bson.M

`bson.M` 是 map：

```go
filter := bson.M{
    "status": "active",
    "age": bson.M{
        "$gte": 18,
    },
}
```

适合：

- 简单过滤。
- 临时构造文档。
- 字段顺序不重要的场景。

注意：Go map 无序。如果字段顺序会影响语义，例如 sort、command、部分复杂聚合，优先用 `bson.D`。

## 4. bson.A

`bson.A` 是数组：

```go
filter := bson.D{
    {"status", bson.D{
        {"$in", bson.A{"active", "blocked"}},
    }},
}
```

也可以写普通 slice：

```go
filter := bson.D{
    {"status", bson.D{
        {"$in", []string{"active", "blocked"}},
    }},
}
```

学习阶段可以两种都见一见；在需要表达 BSON 数组语义时，`bson.A` 更直观。

## 5. bson.ObjectID

模型中 `_id` 通常使用：

```go
bson.ObjectID
```

示例：

```go
type User struct {
    ID        bson.ObjectID `bson:"_id,omitempty" json:"id"`
    Email     string        `bson:"email" json:"email"`
    Name      string        `bson:"name" json:"name"`
    CreatedAt time.Time     `bson:"created_at" json:"created_at"`
    UpdatedAt time.Time     `bson:"updated_at" json:"updated_at"`
}
```

生成新 ObjectID：

```go
id := bson.NewObjectID()
```

从字符串转换：

```go
id, err := bson.ObjectIDFromHex("66a1f1c2e138237e2f7d0001")
if err != nil {
    return err
}
```

转成字符串：

```go
hex := id.Hex()
```

判断是否为零值：

```go
if user.ID.IsZero() {
    user.ID = bson.NewObjectID()
}
```

## 6. Struct Tag

Go 结构体字段需要导出，也就是首字母大写：

```go
type User struct {
    Email string `bson:"email" json:"email"`
}
```

如果字段未导出：

```go
type User struct {
    email string `bson:"email"`
}
```

Go Driver 不会正常编码这个字段。

常见 tag：

```go
ID        bson.ObjectID `bson:"_id,omitempty" json:"id"`
Email     string        `bson:"email" json:"email"`
CreatedAt time.Time     `bson:"created_at" json:"created_at"`
```

含义：

- `bson:"email"`：MongoDB 字段名是 `email`。
- `json:"email"`：API JSON 字段名是 `email`。
- `omitempty`：字段是零值时，编码时可以省略。

## 7. omitempty 的使用

常见场景：

```go
ID bson.ObjectID `bson:"_id,omitempty" json:"id"`
```

插入时如果 `ID` 是零值，驱动或 MongoDB 可以生成 `_id`。

但在业务代码中，很多团队更喜欢自己设置：

```go
if user.ID.IsZero() {
    user.ID = bson.NewObjectID()
}
```

这样插入后业务对象里也立刻有 ID。

不要滥用 `omitempty`。例如必填字段如果被零值省略，可能掩盖数据质量问题。

## 8. 时间字段

```go
type User struct {
    CreatedAt time.Time `bson:"created_at" json:"created_at"`
    UpdatedAt time.Time `bson:"updated_at" json:"updated_at"`
}
```

创建时：

```go
now := time.Now().UTC()
user.CreatedAt = now
user.UpdatedAt = now
```

建议：

- 数据库存 UTC。
- 展示层再按用户时区转换。
- 不要依赖 ObjectID 时间代替业务时间。

## 9. DTO 和数据库模型

数据库模型：

```go
type User struct {
    ID           bson.ObjectID `bson:"_id,omitempty" json:"id"`
    Email        string        `bson:"email" json:"email"`
    Name         string        `bson:"name" json:"name"`
    PasswordHash string        `bson:"password_hash" json:"-"`
    CreatedAt    time.Time     `bson:"created_at" json:"created_at"`
    UpdatedAt    time.Time     `bson:"updated_at" json:"updated_at"`
}
```

API 返回 DTO：

```go
type UserResponse struct {
    ID        string    `json:"id"`
    Email     string    `json:"email"`
    Name      string    `json:"name"`
    CreatedAt time.Time `json:"created_at"`
}
```

转换：

```go
func ToUserResponse(u User) UserResponse {
    return UserResponse{
        ID:        u.ID.Hex(),
        Email:     u.Email,
        Name:      u.Name,
        CreatedAt: u.CreatedAt,
    }
}
```

不要把密码哈希、内部状态、风控字段直接返回给前端。

## 10. 指针字段和局部更新

更新请求中常用指针区分“没传”和“传了零值”：

```go
type UpdateUserInput struct {
    Name *string
    Age  *int
}
```

构造更新：

```go
set := bson.D{}

if input.Name != nil {
    set = append(set, bson.E{Key: "name", Value: *input.Name})
}

if input.Age != nil {
    set = append(set, bson.E{Key: "age", Value: *input.Age})
}
```

如果用普通字段：

```go
type UpdateUserInput struct {
    Name string
    Age  int
}
```

你无法区分用户没传 `age`，还是传了 `age: 0`。

## 11. 常见错误

### BSON 字段名拼错

模型：

```go
Email string `bson:"emial"`
```

数据库里会写成 `emial`，查询 `email` 查不到。

### ObjectID 类型混用

v2 项目里使用：

```go
bson.ObjectID
```

不要混入旧路径中的 `primitive.ObjectID`。

### 把 ObjectID 当字符串查

错误：

```go
filter := bson.D{{"_id", "66a1f1c2e138237e2f7d0001"}}
```

正确：

```go
id, err := bson.ObjectIDFromHex("66a1f1c2e138237e2f7d0001")
if err != nil {
    return err
}

filter := bson.D{{"_id", id}}
```

## 12. 本节验收

你应该能回答：

- `bson.D` 和 `bson.M` 有什么区别？
- 为什么排序建议使用 `bson.D`？
- Go Driver v2 中 ObjectID 类型是什么？
- 如何从 hex 字符串转 ObjectID？
- `bson:"_id,omitempty"` 的作用是什么？
- 为什么 API DTO 和数据库 Model 最好分开？

