# 1. 添加 HTTP 注解与路由设计

本节目标：在 proto 中为 RPC 方法设计清晰的 HTTP/JSON 路由。

HTTP 路由不是随便映射一下就好。对外 API 要符合资源语义，路径、方法、请求体都要稳定。gRPC-Gateway 的注解让这些规则写在 proto 中。

---

## 一、常见路由规则

- 查询资源通常用 GET。
- 创建资源通常用 POST。
- 更新资源可以用 PUT 或 PATCH。
- 删除资源通常用 DELETE。
- 路径参数使用 `{id}` 这类形式。
- POST/PUT/PATCH 可以用 `body` 指定请求体。

---

## 二、引入 annotation

```proto
import "google/api/annotations.proto";
```

示例：

```proto
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse) {
    option (google.api.http) = { get: "/v1/users/{id}" };
  }

  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse) {
    option (google.api.http) = {
      post: "/v1/users"
      body: "*"
    };
  }
}
```

---

## 三、路径设计建议

```text
GET    /v1/users/{id}
POST   /v1/users
GET    /v1/orders/{id}
POST   /v1/orders
GET    /v1/orders/{id}/events
```

路径里带 `/v1` 是为了给未来版本升级留下空间。

---

## 四、常见问题

- 把动作全写进 URL：例如 `/getUser`，不如 `/v1/users/{id}` 稳定。
- 创建接口忘记 `body`：HTTP JSON 无法正确映射请求体。
- 路径没有版本：对外 API 后续升级困难。
- 一个 RPC 映射多个复杂 HTTP 路由：会增加维护成本。

---

## 五、练习任务

1. 为 UserService 的 3 个方法设计 HTTP 路由。
2. 写出每个路由对应的 curl 命令。
3. 判断哪些字段应该出现在 path 中。
4. 给 OrderService 设计创建和查询路由。

---

## 六、完成标准

- proto 中有清晰 HTTP annotation。
- 路由符合基本 REST 直觉。
- curl 请求能映射到 gRPC 方法。


---

## 七、完整操作步骤

本节专注于 proto 中的 HTTP 路由设计。不要急着生成代码，先把路由设计清楚。

操作步骤：

1. 引入 `google/api/annotations.proto`。
2. 给查询接口设计 GET 路由。
3. 给创建接口设计 POST 路由。
4. 明确 path 参数来自哪个 request 字段。
5. 明确 body 映射规则。
6. 为路由加版本号 `/v1`。
7. 检查 HTTP 路由是否暴露了内部实现细节。

---

## 八、完整 proto：GET 路由

```proto
import "google/api/annotations.proto";

message GetUserRequest {
  int64 id = 1;
}

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse) {
    option (google.api.http) = {
      get: "/v1/users/{id}"
    };
  }
}
```

`{id}` 会从 `GetUserRequest.id` 中取值。

HTTP 请求：

```text
GET /v1/users/1
```

会被转换为 gRPC 请求：

```json
{"id": 1}
```

---

## 九、完整 proto：POST 路由

```proto
message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message CreateUserResponse {
  User user = 1;
}

service UserService {
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse) {
    option (google.api.http) = {
      post: "/v1/users"
      body: "*"
    };
  }
}
```

`body: "*"` 表示 HTTP JSON body 整体映射到 `CreateUserRequest`。

HTTP 请求：

```json
{
  "name": "Tom",
  "email": "tom@example.com"
}
```

会映射为：

```proto
CreateUserRequest {
  name: "Tom"
  email: "tom@example.com"
}
```

---

## 十、常见路由设计规范

推荐：

```text
GET    /v1/users/{id}
POST   /v1/users
GET    /v1/orders/{id}
POST   /v1/orders
GET    /v1/orders/{id}/events
```

不推荐：

```text
GET /getUser?id=1
POST /createUser
POST /doOrder
GET /query_user_table
```

HTTP API 面向外部调用方，应该表达资源和动作，而不是暴露内部数据库或 RPC 实现细节。

---

## 十一、body 映射规则

### body: "*"

```proto
option (google.api.http) = {
  post: "/v1/users"
  body: "*"
};
```

整个 JSON body 映射到 request。

### body: "user"

如果 request 是：

```proto
message UpdateUserRequest {
  int64 id = 1;
  User user = 2;
}
```

可以写：

```proto
option (google.api.http) = {
  put: "/v1/users/{id}"
  body: "user"
};
```

此时 path 中的 id 映射到 `id`，body 映射到 `user` 字段。

---

## 十二、运行命令：只验证 proto 编译

写完 annotation 后，先生成代码验证 proto 是否正确：

```powershell
protoc -I . -I third_party `
  --go_out=. --go_opt=paths=source_relative `
  --go-grpc_out=. --go-grpc_opt=paths=source_relative `
  --grpc-gateway_out=. --grpc-gateway_opt=paths=source_relative `
  proto/user/v1/user.proto
```

预期生成：

```text
user.pb.go
user_grpc.pb.go
user.pb.gw.go
```

---

## 十三、常见错误排查

### 1. 路径字段不存在

错误示例：

```proto
get: "/v1/users/{user_id}"
```

但 request 中只有：

```proto
int64 id = 1;
```

这会导致映射失败。路径变量必须能对应 request 字段。

### 2. POST 忘记 body

如果没有写 body，HTTP body 可能不会按预期映射到 request。

### 3. 路由没有版本号

建议对外路由都带 `/v1`，后续升级时更稳。

### 4. JSON 字段名和 proto 字段名混淆

proto 字段 `product_id` 在 JSON 中通常是 `productId`。文档中要写 JSON 形式。

---

## 十四、练习任务

1. 为 `GetUser` 设计 GET 路由。
2. 为 `CreateUser` 设计 POST 路由。
3. 为 `UpdateUser` 设计 PUT 路由。
4. 为 `DeleteUser` 设计 DELETE 路由。
5. 写出每个路由对应的 curl 命令。
6. 检查每个 path 变量都能对应 request 字段。

---

## 十五、完成标准

完成本节后，你应该能：

```text
使用 google.api.http annotation
为 GET/POST/PUT/DELETE 设计路由
解释 body: "*" 和 body: "user" 的区别
避免暴露内部实现细节
为 HTTP API 加版本前缀
写出对应 curl 请求
```

---

## 教程闭环检查

1. **完整操作步骤**：完成 import、GET/POST/PUT/DELETE 注解设计。
2. **完整代码**：使用正文中的 GetUser/CreateUser annotation 示例。
3. **运行命令**：执行 protoc 生成，确认 `.gw.go` 出现。
4. **预期输出**：生成文件包含 `user.pb.gw.go`。
5. **常见错误排查**：重点检查路径字段、body、版本号、JSON 字段名。
6. **练习任务**：完成四类 HTTP 方法路由设计。
7. **完成标准**：能独立为 gRPC RPC 方法设计 HTTP/JSON 路由。