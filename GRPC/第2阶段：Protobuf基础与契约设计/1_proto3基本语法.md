# 1. proto3 基本语法

本节目标：掌握 proto3 文件的基本结构和常用语法。

一个 `.proto` 文件通常包含语法版本、包名、Go 包选项、message、enum 和 service。初学时不要追求复杂特性，先把结构写规范。

---

## 一、核心直觉

- `syntax = "proto3";` 必须放在文件开头附近。
- `package` 是 Protobuf 层面的命名空间，不等于 Go package。
- `option go_package` 是 Go 代码生成所需的路径和包名。
- `message` 里的每个字段都要有类型、名称和编号。
- 字段名称建议使用 snake_case，生成 Go 后会变成驼峰。

---

## 二、动手步骤

1. 创建 `proto/user/v1/user.proto`。
2. 写入 `syntax`、`package`、`go_package`。
3. 定义一个 `User` message。
4. 定义请求响应 message。
5. 定义 service 和 rpc 方法。

---

## 三、参考代码或命令

```proto
syntax = "proto3";

package user.v1;

option go_package = "example.com/grpc-learning/gen/user/v1;userv1";

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
  bool active = 4;
}
```

---

## 四、常见问题

- 字段名写成大驼峰：proto 里推荐 snake_case。
- 忘记字段编号：每个字段都必须有唯一编号。
- 把编号当成排序用：编号是编码标识，不只是显示顺序。

---

## 五、练习任务

- 给 `User` 增加 `created_at` 字段，可以先用 `string` 表示。
- 写一个 `ListUsersRequest` 和 `ListUsersResponse`。
- 尝试把字段顺序调整，观察生成代码是否还能工作。

---

## 六、完成标准

- 能写出结构完整的 proto3 文件。
- 字段命名、编号和包名都符合基本规范。
- 能理解 proto 文件每一部分的作用。


---

## 七、一份 proto 文件从上到下怎么读

以这个文件为例：

```proto
syntax = "proto3";

package user.v1;

option go_package = "example.com/grpc-learning/proto/user/v1;userv1";

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

逐行理解：

- `syntax = "proto3";`：声明使用 proto3 语法。
- `package user.v1;`：Protobuf 层面的命名空间。
- `option go_package = ...;`：Go 代码生成时使用的 import 路径和包名。
- `message`：定义请求或响应结构。
- `service`：定义一组 RPC 方法。
- `rpc`：定义一个具体远程调用。

---

## 八、package 和 go_package 的区别

这是初学者非常容易混淆的地方。

```proto
package user.v1;
option go_package = "example.com/grpc-learning/proto/user/v1;userv1";
```

`package user.v1` 是 Protobuf 的包名。它影响 gRPC 方法全名，例如：

```text
/user.v1.UserService/GetUser
```

`go_package` 是 Go import 路径和 Go 包名。分号前面是 import 路径，分号后面是 Go package 名：

```text
example.com/grpc-learning/proto/user/v1 -> import 路径
userv1 -> Go 包名
```

在 Go 代码里通常这样 import：

```go
import userv1 "example.com/grpc-learning/proto/user/v1"
```

---

## 九、字段编号的基本规则

```proto
message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
}
```

字段编号不是排序装饰。它参与 Protobuf 二进制编码。

规则：

- 同一个 message 内字段编号不能重复。
- 已发布字段编号不要修改。
- 删除字段后不要复用编号。
- 常用字段可以使用较小编号。
- 1 到 15 的编号编码更短，但初学不用过度优化。

---

## 十、命名规范

推荐：

```proto
message GetUserRequest {}
message GetUserResponse {}
service UserService {}
rpc GetUser(...) returns (...);
string created_at = 1;
```

不要写成：

```proto
message get_user_request {}
string CreatedAt = 1;
rpc get_user(...) returns (...);
```

字段名用 snake_case，message、service、rpc 使用大驼峰。生成 Go 代码后字段会自动变成大驼峰。

---

## 十一、动手实验

1. 创建 `proto/user/v1/user.proto`。
2. 写入最小 proto。
3. 执行生成命令。
4. 打开 `user.pb.go`，找到 `type GetUserRequest struct`。
5. 给 `GetUserRequest` 增加 `bool verbose = 2;`。
6. 重新生成代码。
7. 观察 Go 结构体多了什么字段。

生成命令：

```powershell
protoc --go_out=. --go_opt=paths=source_relative `
  --go-grpc_out=. --go-grpc_opt=paths=source_relative `
  proto/user/v1/user.proto
```

---

## 十二、常见错误

### 错误 1：字段编号重复

```proto
message User {
  int64 id = 1;
  string name = 1;
}
```

这会直接编译失败。每个字段编号必须唯一。

### 错误 2：忘记分号

proto 每个字段定义后面都要有分号：

```proto
string name = 1;
```

### 错误 3：package 没有版本

```proto
package user;
```

学习可以运行，但真实项目更建议：

```proto
package user.v1;
```

这样后续 v2 可以并存。

---

## 十三、完成标准

你能独立写出包含下面元素的 proto 文件：

```text
syntax
package
go_package
message
service
rpc
字段类型
字段编号
```
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