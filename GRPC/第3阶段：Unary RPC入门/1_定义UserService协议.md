# 1. 定义 UserService 协议

本节目标：设计第一个可运行的用户服务 proto 契约。

在写 server 之前，先写 proto。gRPC 项目应该优先从契约出发，而不是先写实现再补接口。

---

## 一、核心直觉

- 请求 message 不要直接等同于数据库表。
- 响应 message 可以包装业务对象，方便后续增加 metadata 字段。
- service 方法名要清晰表达业务能力。
- v1 包名给后续版本升级留下空间。

---

## 二、动手步骤

1. 创建 `proto/user/v1/user.proto`。
2. 定义 `User`、`GetUserRequest`、`GetUserResponse`。
3. 定义 `UserService`。
4. 执行生成命令。

---

## 三、参考代码或命令

```proto
syntax = "proto3";

package user.v1;

option go_package = "example.com/grpc-learning/proto/user/v1;userv1";

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
}

message GetUserRequest {
  int64 id = 1;
}

message GetUserResponse {
  User user = 1;
}

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}
```

---

## 四、常见问题

- 返回值直接写 `User` 而不是 `GetUserResponse`：可以工作，但不利于后续扩展响应字段。
- `id` 类型频繁变化：ID 类型应该尽早统一。
- 包名没有版本：后续 v2 升级会比较难安排。

---

## 五、练习任务

- 给 User 增加 `status` 字段。
- 增加 `CreateUser` 的 request/response，但先不实现。
- 重新生成代码。

---

## 六、完成标准

- proto 可以成功生成 Go 文件。
- 能在生成代码里找到 `GetUserRequest` 和 `UserServiceServer`。
- 能解释为什么要有独立 Response。


---

## 七、完整 user.proto

在 `proto/user/v1/user.proto` 中写入：

```proto
syntax = "proto3";

package user.v1;

option go_package = "example.com/grpc-learning/proto/user/v1;userv1";

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
}

message GetUserRequest {
  int64 id = 1;
}

message GetUserResponse {
  User user = 1;
}

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}
```

如果你的 `go.mod` 不是 `example.com/grpc-learning`，请把 `go_package` 前半部分改成你自己的 module 路径。

---

## 八、生成代码

PowerShell：

```powershell
protoc --go_out=. --go_opt=paths=source_relative `
  --go-grpc_out=. --go-grpc_opt=paths=source_relative `
  proto/user/v1/user.proto
```

检查生成文件：

```powershell
Get-ChildItem proto/user/v1
```

预期：

```text
user.proto
user.pb.go
user_grpc.pb.go
```

---

## 九、打开生成代码看三个重点

在 `user.pb.go` 中找：

```go
type GetUserRequest struct
```

你会看到 `Id int64` 字段。

在 `user_grpc.pb.go` 中找：

```go
type UserServiceClient interface
```

它是客户端要使用的接口。

再找：

```go
type UserServiceServer interface
```

它是服务端要实现的接口。

最后找：

```go
func RegisterUserServiceServer
```

它用于把你的 server 实现注册到 gRPC server。

---

## 十、为什么先只写 GetUser

第一个 demo 不要设计太多方法。只写 `GetUser` 的好处是：

- 请求参数只有一个 id。
- 响应结构简单。
- 容易构造内存数据。
- 可以快速跑通 server/client。
- 后续加错误处理时也直观。

等你理解完整流程后，再添加 `CreateUser`、`ListUsers`。

---

## 十一、常见问题

### 生成代码路径和 import 不匹配

如果 server 中 import：

```go
import userv1 "example.com/grpc-learning/proto/user/v1"
```

但 Go 提示找不到包，请检查：

1. `go.mod` 的 module 是否是 `example.com/grpc-learning`。
2. `go_package` 是否一致。
3. 生成文件是否真的在 `proto/user/v1`。

### 忘记生成 gRPC 代码

只执行了：

```powershell
protoc --go_out=. ...
```

但没有执行 `--go-grpc_out`，就不会有 `user_grpc.pb.go`，也就没有 server/client 接口。

---

## 十二、完成标准

你应该已经拥有：

```text
proto/user/v1/user.proto
proto/user/v1/user.pb.go
proto/user/v1/user_grpc.pb.go
```

并能在生成代码中找到：

```text
GetUserRequest
GetUserResponse
UserServiceClient
UserServiceServer
RegisterUserServiceServer
```


---

## 十三、为后续扩展预留空间

虽然本节只实现 `GetUser`，但设计 proto 时可以提前想清楚未来会怎么扩展。

例如后续可能增加：

```proto
message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
}

message ListUsersResponse {
  repeated User users = 1;
  int64 total = 2;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message CreateUserResponse {
  User user = 1;
}
```

但第一轮不要急着全部实现。proto 可以先设计少量未来接口，代码实现按阶段推进。

---

## 十四、字段语义说明

建议在 proto 中写注释：

```proto
message User {
  // 用户唯一 ID。学习阶段使用 int64，真实项目可以替换为雪花 ID 或数据库 ID。
  int64 id = 1;

  // 用户显示名称。
  string name = 2;

  // 用户邮箱，创建用户时需要保证唯一。
  string email = 3;
}
```

注释的价值是让调用方知道字段含义，而不是只看到类型。

---

## 十五、协议设计自检

提交 proto 前，问自己：

1. package 是否带版本？
2. go_package 是否和 go.mod 对应？
3. 请求和响应是否分开？
4. 字段编号是否从 1 开始且不重复？
5. 字段名是否是 snake_case？
6. service 名是否表达业务边界？
7. rpc 名是否表达业务动作？

如果这些都没问题，再生成代码。

---

## 十六、下一节衔接

下一节服务端要做的事情，就是实现生成代码中的这个接口：

```go
type UserServiceServer interface {
    GetUser(context.Context, *GetUserRequest) (*GetUserResponse, error)
}
```

所以你现在不只是写了一个配置文件，而是定义了后面 Go 代码必须遵守的接口。
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