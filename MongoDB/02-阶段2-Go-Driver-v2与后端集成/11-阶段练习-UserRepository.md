# 11. 阶段练习：UserRepository

这是第 2 阶段的综合练习。你要完成一个可测试的 `UserRepository`，把本阶段所有内容串起来。

## 1. 练习目标

你需要实现：

- MongoDB 连接初始化。
- 用户模型。
- Repository 封装。
- 创建用户。
- 按 ID 查询用户。
- 按邮箱查询用户。
- 用户列表查询。
- 更新用户资料。
- 增加和移除角色。
- 软删除用户。
- 错误转换。
- 集成测试。

## 2. 目录结构

```text
mongodb-go-stage2/
  cmd/
    api/
      main.go
  internal/
    config/
      config.go
    database/
      mongo.go
    user/
      model.go
      errors.go
      repository.go
      repository_test.go
```

## 3. 必须实现的方法

```go
func NewRepository(db *mongo.Database) *Repository

func (r *Repository) Create(ctx context.Context, u *User) error

func (r *Repository) FindByID(ctx context.Context, id bson.ObjectID) (*User, error)

func (r *Repository) FindByEmail(ctx context.Context, email string) (*User, error)

func (r *Repository) List(ctx context.Context, input ListFilter) ([]User, error)

func (r *Repository) Count(ctx context.Context, input ListFilter) (int64, error)

func (r *Repository) UpdateProfile(ctx context.Context, id bson.ObjectID, input UpdateProfileInput) error

func (r *Repository) AddRole(ctx context.Context, id bson.ObjectID, role string) error

func (r *Repository) RemoveRole(ctx context.Context, id bson.ObjectID, role string) error

func (r *Repository) SoftDelete(ctx context.Context, id bson.ObjectID) error
```

## 4. 必须定义的错误

```go
var (
    ErrNotFound      = errors.New("user not found")
    ErrEmailExists   = errors.New("email already exists")
    ErrInvalidUpdate = errors.New("invalid update")
)
```

要求：

- `FindByID` 和 `FindByEmail` 查不到时返回 `ErrNotFound`。
- `Create` 邮箱重复时返回 `ErrEmailExists`。
- `UpdateProfile` 没有任何字段可更新时返回 `ErrInvalidUpdate`。
- `UpdateProfile`、`AddRole`、`RemoveRole`、`SoftDelete` 找不到用户时返回 `ErrNotFound`。

## 5. 必须创建的索引

在 `mongosh` 中：

```javascript
use go_mongodb_stage2
db.users.createIndex({ email: 1 }, { unique: true })
db.users.createIndex({ status: 1, created_at: -1 })
db.users.createIndex({ roles: 1 })
```

第 2 阶段先了解这些索引即可。第 5 阶段会系统学习索引设计和 explain。

## 6. main.go 验证流程

`main.go` 至少完成：

```text
1. 读取配置
2. 连接 MongoDB
3. 创建 UserRepository
4. 创建一个用户
5. 按邮箱查询用户
6. 更新用户资料
7. 增加角色
8. 查询列表
9. 软删除用户
10. 再次查询确认返回 ErrNotFound
```

你不需要接 HTTP，但输出要能看出每一步结果。

## 7. 测试要求

至少写 6 个测试：

```text
TestRepository_CreateAndFindByEmail
TestRepository_FindByEmail_NotFound
TestRepository_Create_DuplicateEmail
TestRepository_List
TestRepository_UpdateProfile
TestRepository_SoftDelete
```

如果时间充足，再加：

```text
TestRepository_AddRole
TestRepository_RemoveRole
TestRepository_UpdateProfile_InvalidUpdate
```

运行：

```powershell
go test ./...
```

## 8. 代码质量要求

必须做到：

- Repository 方法都接收 `context.Context`。
- 不在 Repository 里使用 `context.Background()` 执行业务操作。
- 不在每个方法里创建 `mongo.Client`。
- `Find` 查询必须关闭 Cursor。
- `Find` 遍历后必须检查 `cursor.Err()`。
- 所有列表查询必须有 limit。
- 所有软删除敏感查询必须过滤 `deleted_at`。
- 不把 `mongo.ErrNoDocuments` 直接暴露给上层。
- 不把重复键错误直接暴露给上层。

## 9. 自查问题

请回答：

1. 为什么一个服务通常只创建一个 `mongo.Client`？
2. `mongo.Connect` 成功是否代表 MongoDB 一定可达？
3. 为什么还要 `Ping`？
4. `bson.D` 和 `bson.M` 怎么选？
5. Go Driver v2 中 ObjectID 用什么类型？
6. `FindOne` 没查到时什么地方返回错误？
7. `Find` 为什么要关闭 Cursor？
8. `MatchedCount = 1` 且 `ModifiedCount = 0` 是否一定是失败？
9. 软删除会给查询带来什么额外要求？
10. 为什么 Repository 测试建议连真实 MongoDB？

## 10. 阶段验收标准

满足下面条件，可以进入第 3 阶段：

- 能从零创建 Go module 并安装 Go Driver v2。
- 能正确导入 v2 包。
- 能连接 MongoDB 并 Ping。
- 能解释 Client、Database、Collection 的关系。
- 能定义包含 `bson.ObjectID` 的模型。
- 能使用 BSON tag。
- 能实现 InsertOne、FindOne、Find、UpdateOne、DeleteOne。
- 能正确处理 Cursor。
- 能处理未找到、重复键、超时错误。
- 能写 Repository 集成测试。
- 能完成 `UserRepository` 综合练习。

第 3 阶段将进入文档建模：嵌入还是引用、一对多、多对多、反范式、数组增长、数据生命周期和 Schema Validation。那一阶段会决定你能不能真正把 MongoDB 用得像一个后端工程师，而不是只会写 CRUD。

