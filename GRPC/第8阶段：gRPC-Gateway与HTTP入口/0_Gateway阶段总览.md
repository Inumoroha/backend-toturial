# 0. gRPC-Gateway 阶段总览

本阶段目标：理解内部 gRPC 服务如何对外提供 HTTP/JSON API。

很多系统内部使用 gRPC，但外部用户、浏览器、第三方平台仍然更习惯 HTTP/JSON。gRPC-Gateway 可以把 HTTP 请求转成 gRPC 调用，让你用一份 proto 同时服务内部和外部入口。

---

## 一、Gateway 在系统里的位置

```text
Browser / Third-party Client
        |
        v
HTTP/JSON API Gateway
        |
        v
gRPC internal services
```

Gateway 不应该承载复杂业务。它主要负责：协议转换、基础鉴权、请求日志、错误映射、跨域处理等入口层能力。

---

## 二、本阶段学习内容

- 在 proto 中添加 HTTP annotation。
- 生成 `.gw.pb.go` 代码。
- 启动 HTTP gateway server。
- 处理 gRPC code 到 HTTP status 的映射。
- 生成 OpenAPI 文档。

---

## 三、基本路由示例

```proto
rpc GetUser(GetUserRequest) returns (GetUserResponse) {
  option (google.api.http) = {
    get: "/v1/users/{id}"
  };
}
```

HTTP 调用：

```bash
curl http://localhost:8080/v1/users/1
```

内部仍然调用：

```text
user.v1.UserService/GetUser
```

---

## 四、常见问题

- 以为 Gateway 替代 gRPC server：Gateway 只是转发入口。
- 对外暴露内部字段：HTTP API 要更谨慎地设计返回。
- 忽略错误映射：HTTP 调用方看不懂 gRPC code。
- Gateway 层写太多业务：会让入口层变成新的单体。

---

## 五、完成标准

- 能解释 Gateway 的位置。
- 能完成一个 HTTP 到 gRPC 的转发。
- 知道对外 API 设计要更谨慎。


---

## 七、为什么需要 gRPC-Gateway

前面阶段的 gRPC 服务更适合内部服务之间调用，但真实后端系统经常还有这些需求：

```text
浏览器要调用 API
前端同学希望拿到 JSON
第三方系统不会接 gRPC
测试人员习惯 curl/Postman
需要生成 OpenAPI 文档
```

这时通常不会把内部 gRPC 全部改成 HTTP，而是在外面加一层 HTTP/JSON Gateway：

```text
Browser / Third-party / curl
        |
        v
HTTP/JSON Gateway
        |
        v
gRPC UserService / OrderService / ProductService
```

Gateway 的职责是协议转换，不是重新写一套业务逻辑。

---

## 八、本阶段实验结构

继续使用 `grpc-learning`：

```text
grpc-learning/
  proto/user/v1/user.proto
  proto/user/v1/user.pb.go
  proto/user/v1/user_grpc.pb.go
  proto/user/v1/user.pb.gw.go
  cmd/server/main.go
  cmd/gateway/main.go
  openapi/
    user.swagger.json
```

本阶段新增：

- `user.pb.gw.go`：gRPC-Gateway 生成代码。
- `cmd/gateway/main.go`：HTTP gateway 入口。
- `openapi/`：OpenAPI 文档输出目录。

---

## 九、本阶段完整操作步骤

1. 安装 gateway 生成插件。
2. 在 proto 中引入 `google/api/annotations.proto`。
3. 给 `GetUser` 添加 HTTP GET 路由。
4. 给 `CreateUser` 添加 HTTP POST 路由。
5. 生成 Go message、gRPC、Gateway 三类代码。
6. 启动 gRPC server。
7. 启动 HTTP gateway。
8. 使用 curl 调用 HTTP API。
9. 观察 gRPC status 如何映射成 HTTP status。
10. 生成 OpenAPI 文档。

---

## 十、完整 proto 示例

```proto
syntax = "proto3";

package user.v1;

option go_package = "example.com/grpc-learning/proto/user/v1;userv1";

import "google/api/annotations.proto";

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

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message CreateUserResponse {
  User user = 1;
}

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse) {
    option (google.api.http) = {
      get: "/v1/users/{id}"
    };
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

## 十一、生成代码命令

安装插件：

```powershell
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway@latest
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2@latest
```

生成 Go 和 Gateway：

```powershell
protoc -I . -I third_party `
  --go_out=. --go_opt=paths=source_relative `
  --go-grpc_out=. --go-grpc_opt=paths=source_relative `
  --grpc-gateway_out=. --grpc-gateway_opt=paths=source_relative `
  proto/user/v1/user.proto
```

说明：`third_party` 用来放 `google/api/annotations.proto` 等依赖文件。如果你使用 buf 或依赖管理工具，路径会不同。

---

## 十二、运行命令

终端 1 启动 gRPC server：

```powershell
go run ./cmd/server
```

终端 2 启动 HTTP gateway：

```powershell
go run ./cmd/gateway
```

终端 3 调用 HTTP API：

```powershell
curl http://localhost:8080/v1/users/1
```

预期输出：

```json
{
  "user": {
    "id": "1",
    "name": "Tom",
    "email": "tom@example.com"
  }
}
```

---

## 十三、常见错误排查

### 1. annotations.proto 找不到

错误类似：

```text
google/api/annotations.proto: File not found.
```

解决：确认 `-I` include 路径中包含 `google/api/annotations.proto` 所在目录。

### 2. 没有生成 .gw.go

检查是否安装：

```powershell
Get-Command protoc-gen-grpc-gateway
```

以及生成命令里是否有：

```text
--grpc-gateway_out
```

### 3. Gateway 启动了但请求失败

检查 gRPC server 是否启动，gateway 是否连接到正确地址，例如 `localhost:50051`。

### 4. 字段 JSON 名不符合预期

Protobuf JSON 默认会把 `user_id` 转成 `userId`。对外 API 文档要写清楚。

---

## 十四、练习任务

1. 为 `GetUser` 添加 HTTP GET 路由。
2. 为 `CreateUser` 添加 HTTP POST 路由。
3. 生成 `.gw.go` 文件。
4. 启动 gRPC server 和 HTTP gateway。
5. 使用 curl 调用 `/v1/users/1`。
6. 触发 NotFound，观察 HTTP status。

---

## 十五、完成标准

完成本阶段总览后，你应该能：

```text
解释 gRPC-Gateway 的作用
在 proto 中添加 HTTP annotation
生成 gateway 代码
启动 HTTP gateway server
用 curl 调用内部 gRPC 服务
理解 gRPC code 到 HTTP status 的映射
知道 OpenAPI 文档从哪里生成
```

---

## 教程闭环检查

1. **完整操作步骤**：完成 annotation、代码生成、server/gateway 启动、curl 调用。
2. **完整代码**：使用正文中的 proto 和后续小节的 gateway main 示例。
3. **运行命令**：执行 protoc、go run server、go run gateway、curl。
4. **预期输出**：curl 能拿到 JSON 用户响应。
5. **常见错误排查**：重点检查 annotations.proto、插件、gRPC server 地址、JSON 字段名。
6. **练习任务**：完成 GET、POST、NotFound 三类 HTTP 调用。
7. **完成标准**：能把一个内部 gRPC 方法暴露为 HTTP/JSON API。