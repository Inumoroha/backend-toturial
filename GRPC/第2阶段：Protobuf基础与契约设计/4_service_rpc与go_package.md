# 4. service、rpc 与 go_package

本节目标：掌握服务定义、方法定义，以及 Go 生成代码的包路径规则。

message 定义数据，service 定义能力。gRPC 的服务接口来自 proto 中的 `service` 和 `rpc`，Go 代码生成后会变成 client interface、server interface 和注册函数。

---

## 一、核心直觉

- `service UserService` 会生成 `UserServiceClient` 和 `UserServiceServer`。
- Unary RPC 写法是 `rpc GetUser(GetUserRequest) returns (GetUserResponse);`。
- `go_package` 通常写成 `模块路径/生成目录;包名`。
- 包名建议带版本，例如 `userv1`，避免不同版本冲突。
- service 方法名应该表达业务动作，而不是数据库操作。

---

## 二、动手步骤

1. 在 `user.proto` 中定义 `UserService`。
2. 添加 `GetUser`、`CreateUser`、`ListUsers`。
3. 生成 Go 代码。
4. 在 `_grpc.pb.go` 中找到 client、server、register 三类代码。

---

## 三、参考代码或命令

```proto
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
}
```

生成后重点找：

```text
UserServiceClient
UserServiceServer
RegisterUserServiceServer
```

---

## 四、常见问题

- `go_package` 路径和实际目录完全不匹配：会导致 import 混乱。
- service 方法设计得太细：例如 `QueryUserTable` 暴露了数据库实现细节。
- 请求响应 message 混用：创建、查询、列表最好有各自明确的 request/response。

---

## 五、练习任务

- 给 `ProductService` 设计 3 个 RPC 方法。
- 为每个 RPC 单独设计 Request 和 Response。
- 说明这些方法为什么属于 service，而不是普通 message。

---

## 六、完成标准

- 能读懂 service/rpc 定义。
- 能在生成代码里找到关键接口。
- 能合理编写 `go_package`。


---

## 七、service 生成了什么

这段 proto：

```proto
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}
```

会在 `_grpc.pb.go` 中生成几类重要代码：

```go
type UserServiceClient interface {
    GetUser(ctx context.Context, in *GetUserRequest, opts ...grpc.CallOption) (*GetUserResponse, error)
}

type UserServiceServer interface {
    GetUser(context.Context, *GetUserRequest) (*GetUserResponse, error)
    mustEmbedUnimplementedUserServiceServer()
}

func RegisterUserServiceServer(s grpc.ServiceRegistrar, srv UserServiceServer)
```

后面写服务端时，你要实现 `UserServiceServer`。写客户端时，你要使用 `UserServiceClient`。

---

## 八、为什么 request/response 要单独建

有些初学者会想：

```proto
rpc CreateUser(User) returns (User);
```

这种写法短期省事，但不推荐。原因是：

- 创建用户时客户端不应该传 id。
- 返回用户时可能包含 created_at。
- 请求和响应的字段语义不同。
- 后续扩展分页、trace、额外信息会不方便。

更推荐：

```proto
message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message CreateUserResponse {
  User user = 1;
}

rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
```

---

## 九、RPC 方法命名建议

推荐：

```proto
rpc GetUser(GetUserRequest) returns (GetUserResponse);
rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
rpc UpdateUser(UpdateUserRequest) returns (UpdateUserResponse);
rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
```

不推荐：

```proto
rpc QueryUserTable(...);
rpc InsertUserRow(...);
rpc DoUser(...);
rpc Handle(...);
```

RPC 方法应该表达业务能力，而不是数据库实现细节。

---

## 十、go_package 实战规则

如果 `go.mod` 是：

```text
module example.com/grpc-learning
```

proto 文件在：

```text
proto/user/v1/user.proto
```

推荐：

```proto
option go_package = "example.com/grpc-learning/proto/user/v1;userv1";
```

含义：

```text
example.com/grpc-learning/proto/user/v1  -> import 路径
userv1                                   -> Go 包名
```

Go 中使用：

```go
import userv1 "example.com/grpc-learning/proto/user/v1"
```

---

## 十一、完整服务设计练习

为商品服务设计：

```proto
service ProductService {
  rpc GetProduct(GetProductRequest) returns (GetProductResponse);
  rpc ListProducts(ListProductsRequest) returns (ListProductsResponse);
  rpc DeductStock(DeductStockRequest) returns (DeductStockResponse);
}
```

思考：

- `DeductStock` 为什么不叫 `UpdateProduct`？
- `ListProductsResponse` 为什么用 `repeated Product products`？
- `DeductStock` 是否能随意重试？

---

## 十二、常见错误

### go_package 包名不清楚

不推荐：

```proto
option go_package = "example.com/grpc-learning/proto/user/v1";
```

虽然也可能生成，但包名由工具推断，不如显式写清楚：

```proto
option go_package = "example.com/grpc-learning/proto/user/v1;userv1";
```

### service 名过于宽泛

```proto
service CommonService {}
```

这类名字容易变成万能服务。更好的方式是按业务边界拆：

```proto
service UserService {}
service OrderService {}
service ProductService {}
```

---

## 十三、完成标准

你应该能做到：

- 根据业务边界设计 service。
- 给每个 RPC 设计独立 request/response。
- 写出正确的 `go_package`。
- 在生成代码中找到 client、server、register 函数。
---

## 教程闭环检查

为了保证本节不是只停留在概念层面，学习时请按下面闭环完成：

1. **完整操作步骤**：先按正文顺序完成本节涉及的环境检查、文件创建、proto 编写、代码生成或运行验证。
2. **完整代码或命令**：本节如果涉及代码，请使用正文中的完整示例；如果是概念准备章节，请至少执行或记录正文给出的检查命令。
3. **运行命令**：把本节出现的关键命令实际运行一遍，例如 `go version`、`protoc --version`、`protoc ...`、`go run ...` 或 `grpcurl ...`。
4. **预期输出**：运行后对照正文中的预期输出；如果输出不同，先不要跳到下一节。
5. **常见错误排查**：遇到 PATH、go_package、端口占用、连接失败、生成文件缺失等问题时，优先按本节排错思路定位。
6. **练习任务**：完成正文中的练习，不只复制代码，要至少做一次参数或字段修改。
7. **完成标准**：能不看教程复述本节做了什么、为什么这样做、出错时从哪里查，才算真正完成。