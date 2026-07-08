# 0. Protobuf 阶段总览

本节目标：掌握 `.proto` 文件的基本写法，并形成接口契约优先的开发习惯。

在 gRPC 项目里，`.proto` 是服务端和客户端共同遵守的契约。你可以把它理解成比 REST 文档更严格的接口文档：它既给人看，也给工具生成代码。

本阶段目标不是背完所有 Protobuf 语法，而是掌握最常用、最容易踩坑、最影响后续演进的部分。

---

## 一、核心直觉

- message 定义数据结构。
- service 定义服务和方法。
- rpc 定义请求和响应类型。
- 字段编号是二进制编码的一部分，不能随便改。
- `go_package` 决定生成 Go 代码的 import 路径和包名。

---

## 二、动手步骤

1. 从最小 `user.proto` 开始。
2. 先定义请求响应，再定义 service。
3. 执行 `protoc` 生成代码。
4. 阅读生成文件，不要求逐行看懂，但要知道里面有什么。
5. 修改 proto 后重新生成，观察 Go 代码变化。

---

## 三、参考代码或命令

```proto
syntax = "proto3";

package user.v1;

option go_package = "example.com/grpc-learning/gen/user/v1;userv1";

message GetUserRequest {
  int64 id = 1;
}

message GetUserResponse {
  int64 id = 1;
  string name = 2;
}

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}
```

---

## 四、常见问题

- 忽略 `go_package`：生成 Go 代码时很容易出现 import 路径混乱。
- 请求和响应直接复用业务实体：短期省事，长期容易限制接口演进。
- 随便修改字段编号：这会破坏已有客户端兼容性。

---

## 五、练习任务

- 写一个 `user.proto`。
- 加一个 `CreateUser` 方法。
- 加一个 `UserStatus` enum。
- 重新生成代码并观察变化。

---

## 六、完成标准

- 能独立写出简单 proto 文件。
- 能解释 message、service、rpc 的区别。
- 知道字段编号不能乱动。


---

## 七、本阶段的学习方式

Protobuf 不适合只背语法。更好的方式是：

```text
写一个 message
-> 生成 Go 代码
-> 看生成结构体
-> 修改字段
-> 再生成
-> 观察变化
```

你要把 `.proto` 看成接口源代码，而不是配置文件。后续 server 和 client 都围绕它展开。

---

## 八、建议目录

在第 1 阶段创建的工程里继续操作：

```text
grpc-learning/
  proto/
    user/
      v1/
        user.proto
  cmd/
    server/
    client/
  go.mod
```

本阶段只需要写 `proto/user/v1/user.proto`，暂时不写 server 和 client。

---

## 九、第一份完整 proto

```proto
syntax = "proto3";

package user.v1;

option go_package = "example.com/grpc-learning/proto/user/v1;userv1";

enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_DISABLED = 2;
}

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
  UserStatus status = 4;
  repeated string roles = 5;
}

message GetUserRequest {
  int64 id = 1;
}

message GetUserResponse {
  User user = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message CreateUserResponse {
  User user = 1;
}

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
}
```

这份 proto 已经包含：包名、Go 包路径、枚举、message、repeated、service、rpc。

---

## 十、生成代码后要看什么

执行生成命令后，你会看到两个文件：

```text
user.pb.go
user_grpc.pb.go
```

重点看：

```text
user.pb.go:
- type User struct
- type GetUserRequest struct
- type UserStatus int32
- getter 方法，例如 GetId、GetName

user_grpc.pb.go:
- type UserServiceClient interface
- type UserServiceServer interface
- func RegisterUserServiceServer
- type UnimplementedUserServiceServer
```

不要求你逐行理解生成代码，但必须知道这些名字后面会在哪里用。

---

## 十一、阶段完成后你应该具备的能力

完成本阶段后，你应该能独立完成：

1. 创建一个带版本的 proto 包。
2. 为 Go 项目写正确的 `go_package`。
3. 设计 request 和 response。
4. 使用 enum 表示状态。
5. 使用 repeated 表示列表。
6. 使用 reserved 避免字段编号复用。
7. 成功生成 Go 代码。
8. 在生成代码中找到 client/server 接口。

如果这些还不熟，不建议急着进入 server 实现。因为后面所有代码都依赖这一层契约。


---

## 十二、Protobuf 学习的三个层次

学习 Protobuf 可以分成三个层次。

### 第一层：能写

你能写出 message、service、rpc，能执行 protoc 生成代码。这是入门要求。

### 第二层：写得稳

你知道字段编号不能乱改，知道删除字段要 reserved，知道 request 和 response 不要偷懒复用。这是工程要求。

### 第三层：能演进

你知道什么时候继续在 v1 增加字段，什么时候要开 v2；知道如何评估一次 proto 变更会不会影响旧客户端。这是生产要求。

本阶段至少要达到第二层，后续项目实战中逐步进入第三层。

---

## 十三、推荐练习顺序

请按下面顺序练习，不要跳着做：

1. 写最小 `GetUser` proto。
2. 生成 Go 代码。
3. 观察 `GetUserRequest` 结构体。
4. 增加 enum `UserStatus`。
5. 重新生成代码。
6. 增加 `ListUsersResponse`，使用 repeated。
7. 删除一个字段并 reserved。
8. 给 service 增加 `CreateUser`。
9. 观察 `_grpc.pb.go` 中 client/server 接口变化。

这个过程会让你真正理解“proto 驱动代码生成”，而不是只记住命令。

---

## 十四、本阶段不深入的内容

为了保持学习节奏，本阶段暂时不深入：

- `oneof`
- `Any`
- `FieldMask`
- 自定义 option
- buf generate 复杂配置
- OpenAPI annotation

这些内容后面遇到具体场景再学。初学阶段先把最常用的 20% 学扎实。
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