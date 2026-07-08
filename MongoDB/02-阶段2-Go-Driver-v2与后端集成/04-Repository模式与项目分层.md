# 04. Repository 模式与项目分层

本节先不急着写 CRUD，而是把 Go 后端项目中的分层关系搭起来。MongoDB 查询如果散落在 Handler、Service、定时任务里，项目很快会失控。

## 1. 推荐分层

一个简单 Go 后端模块可以这样分：

```text
handler.go      HTTP 参数解析和响应
service.go      业务规则、事务、幂等
repository.go   数据库访问
model.go        数据库模型
dto.go          API 输入输出结构
errors.go       业务错误
```

本阶段聚焦 Repository，所以暂时不写 Handler。

## 2. Repository 负责什么

Repository 负责：

- 构造 MongoDB filter。
- 构造 MongoDB update。
- 调用 Collection 方法。
- Decode 查询结果。
- 处理 Cursor。
- 把 MongoDB 错误转换成更上层能理解的错误。

Repository 不负责：

- 解析 HTTP 请求。
- 判断登录权限。
- 返回 HTTP 状态码。
- 组织复杂业务流程。
- 直接打印响应 JSON。

## 3. User 模型

创建 `internal/user/model.go`：

```go
package user

import (
    "time"

    "go.mongodb.org/mongo-driver/v2/bson"
)

type User struct {
    ID        bson.ObjectID `bson:"_id,omitempty" json:"id"`
    Email     string        `bson:"email" json:"email"`
    Name      string        `bson:"name" json:"name"`
    Age       int           `bson:"age" json:"age"`
    Roles     []string      `bson:"roles" json:"roles"`
    Status    string        `bson:"status" json:"status"`
    CreatedAt time.Time     `bson:"created_at" json:"created_at"`
    UpdatedAt time.Time     `bson:"updated_at" json:"updated_at"`
    DeletedAt *time.Time    `bson:"deleted_at,omitempty" json:"deleted_at,omitempty"`
}
```

为什么 `DeletedAt` 用指针？

- `nil` 表示未删除。
- 非 nil 表示删除时间。
- `omitempty` 可以避免未删除文档写入 `deleted_at: null`。

## 4. Repository 结构

创建 `internal/user/repository.go`：

```go
package user

import "go.mongodb.org/mongo-driver/v2/mongo"

type Repository struct {
    collection *mongo.Collection
}

func NewRepository(db *mongo.Database) *Repository {
    return &Repository{
        collection: db.Collection("users"),
    }
}
```

为什么保存 `*mongo.Collection`？

- Repository 只关心自己的集合。
- 不需要每个方法都传 database。
- 便于测试时注入测试数据库。

## 5. Repository 方法设计

建议先实现：

```go
func (r *Repository) Create(ctx context.Context, u *User) error
func (r *Repository) FindByID(ctx context.Context, id bson.ObjectID) (*User, error)
func (r *Repository) FindByEmail(ctx context.Context, email string) (*User, error)
func (r *Repository) List(ctx context.Context, filter ListFilter) ([]User, error)
func (r *Repository) UpdateProfile(ctx context.Context, id bson.ObjectID, input UpdateProfileInput) error
func (r *Repository) SoftDelete(ctx context.Context, id bson.ObjectID) error
```

注意：

- 方法参数传 `context.Context`。
- 查询单个用户时传 `bson.ObjectID`，不要在 Repository 内到处解析字符串。
- 字符串 ID 转 ObjectID 可以放在 Handler 或 Service 层。
- Repository 返回业务层能理解的错误。

## 6. 输入结构

```go
type ListFilter struct {
    Status string
    Role   string
    Limit  int64
    Offset int64
}

type UpdateProfileInput struct {
    Name *string
    Age  *int
}
```

`UpdateProfileInput` 使用指针，避免把没传的字段更新成零值。

## 7. Repository 是否要定义接口

有两种写法。

### 先写具体类型

```go
repo := user.NewRepository(db)
```

优点：

- 简单。
- 适合学习阶段。
- 不过早抽象。

### Service 层定义接口

```go
type UserRepository interface {
    Create(ctx context.Context, u *User) error
    FindByEmail(ctx context.Context, email string) (*User, error)
}
```

优点：

- 方便 mock。
- Service 不依赖具体 MongoDB 实现。

建议：学习阶段 Repository 先写具体类型；进入 Service 测试时，再按需要抽接口。

## 8. 错误放在哪里

创建 `internal/user/errors.go`：

```go
package user

import "errors"

var (
    ErrNotFound      = errors.New("user not found")
    ErrEmailExists   = errors.New("email already exists")
    ErrInvalidUpdate = errors.New("invalid update")
)
```

Repository 中不要把 `mongo.ErrNoDocuments` 直接丢给上层。上层不应该关心 MongoDB 的细节。

例如：

```go
if errors.Is(err, mongo.ErrNoDocuments) {
    return nil, ErrNotFound
}
```

## 9. 本阶段的最小架构

```text
main.go
  -> config.Load()
  -> database.Connect()
  -> client.Database(...)
  -> user.NewRepository(db)
  -> repo.Create(...)
  -> repo.FindByEmail(...)
```

这是一个没有 HTTP 的小型命令行程序，但结构已经接近真实项目。

后面要接 HTTP，只是多加：

```text
handler -> service -> repository
```

而不是推翻重写。

## 10. 本节验收

你应该能回答：

- Repository 负责什么？
- Repository 不应该负责什么？
- 为什么 Repository 保存 `*mongo.Collection`？
- 为什么所有方法都传 `context.Context`？
- 为什么字符串 ID 不建议在 Repository 到处解析？
- 为什么业务错误要和 MongoDB 底层错误隔离？

