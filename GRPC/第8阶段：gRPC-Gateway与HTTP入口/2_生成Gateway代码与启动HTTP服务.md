# 2. 生成 Gateway 代码与启动 HTTP 服务

本节目标：生成 gRPC-Gateway 代码，并启动一个 HTTP 入口服务。

Gateway 代码同样由 proto 生成。它负责接收 HTTP 请求，解析 JSON 和路径参数，然后调用后端 gRPC client。

---

## 一、安装生成插件

```bash
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway@latest
```

确认插件在 PATH 中：

```powershell
Get-Command protoc-gen-grpc-gateway
```

---

## 二、生成代码

```bash
protoc -I . \
  --grpc-gateway_out=. --grpc-gateway_opt=paths=source_relative \
  proto/user/v1/user.proto
```

PowerShell：

```powershell
protoc -I . `
  --grpc-gateway_out=. --grpc-gateway_opt=paths=source_relative `
  proto/user/v1/user.proto
```

生成后通常会看到 `.gw.pb.go` 文件。

---

## 三、启动 HTTP Gateway

```go
mux := runtime.NewServeMux()
err := userv1.RegisterUserServiceHandlerFromEndpoint(
    ctx,
    mux,
    "localhost:50051",
    []grpc.DialOption{grpc.WithTransportCredentials(insecure.NewCredentials())},
)
if err != nil {
    log.Fatal(err)
}

log.Println("HTTP gateway listening on :8080")
log.Fatal(http.ListenAndServe(":8080", mux))
```

---

## 四、调用 HTTP API

```bash
curl http://localhost:8080/v1/users/1
```

如果服务端返回 gRPC `NotFound`，Gateway 会映射成 HTTP 404。

---

## 五、常见问题

- 忘记 `-I` include 路径：annotations proto 找不到。
- gRPC server 没启动就启动 gateway：HTTP 请求会转发失败。
- gateway 和 server 使用不同 proto 版本：行为可能不一致。
- Gateway 连接地址写错：HTTP 服务启动了但请求全部失败。

---

## 六、练习任务

1. 生成 `.gw.pb.go` 文件。
2. 启动 gRPC server 和 HTTP gateway。
3. 使用 curl 调用 `/v1/users/1`。
4. 触发 NotFound，观察 HTTP 状态码。

---

## 七、完成标准

- Gateway 可以成功启动。
- HTTP 请求能转到 gRPC 服务。
- 你知道 `.gw.pb.go` 的作用。


---

## 七、完整操作步骤

本节要真正启动一个 HTTP gateway，让 `curl http://localhost:8080/v1/users/1` 转发到内部 gRPC `UserService.GetUser`。

操作步骤：

1. 安装 gateway 插件。
2. 生成 `.gw.go` 文件。
3. 创建 `cmd/gateway/main.go`。
4. 在 gateway 中创建 `runtime.ServeMux`。
5. 注册 `UserServiceHandlerFromEndpoint`。
6. 启动 HTTP server 监听 `:8080`。
7. 同时启动 gRPC server 和 gateway。
8. 使用 curl 调用 HTTP API。

---

## 八、安装插件

```powershell
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway@latest
```

验证：

```powershell
Get-Command protoc-gen-grpc-gateway
```

如果找不到，临时加入 PATH：

```powershell
$env:Path += ";$(go env GOPATH)\bin"
```

---

## 九、生成 Gateway 代码

```powershell
protoc -I . -I third_party `
  --go_out=. --go_opt=paths=source_relative `
  --go-grpc_out=. --go-grpc_opt=paths=source_relative `
  --grpc-gateway_out=. --grpc-gateway_opt=paths=source_relative `
  proto/user/v1/user.proto
```

检查：

```powershell
Get-ChildItem proto/user/v1
```

预期：

```text
user.proto
user.pb.go
user_grpc.pb.go
user.pb.gw.go
```

---

## 十、完整代码：cmd/gateway/main.go

```go
package main

import (
    "context"
    "log"
    "net/http"

    userv1 "example.com/grpc-learning/proto/user/v1"
    "github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
)

func main() {
    ctx := context.Background()
    mux := runtime.NewServeMux()

    err := userv1.RegisterUserServiceHandlerFromEndpoint(
        ctx,
        mux,
        "localhost:50051",
        []grpc.DialOption{grpc.WithTransportCredentials(insecure.NewCredentials())},
    )
    if err != nil {
        log.Fatalf("register gateway handler: %v", err)
    }

    log.Println("HTTP gateway listening on :8080")
    if err := http.ListenAndServe(":8080", mux); err != nil {
        log.Fatalf("http serve: %v", err)
    }
}
```

这个 gateway 会把 HTTP 请求转发到 `localhost:50051` 上的 gRPC server。

---

## 十一、安装 Go 依赖

执行：

```powershell
go get github.com/grpc-ecosystem/grpc-gateway/v2/runtime
go mod tidy
```

如果编译提示缺少 gateway 相关包，通常 `go mod tidy` 可以解决。

---

## 十二、运行命令

终端 1：启动 gRPC server。

```powershell
go run ./cmd/server
```

终端 2：启动 HTTP gateway。

```powershell
go run ./cmd/gateway
```

预期输出：

```text
HTTP gateway listening on :8080
```

终端 3：curl 调用。

```powershell
curl http://localhost:8080/v1/users/1
```

预期输出：

```json
{"user":{"id":"1","name":"Tom","email":"tom@example.com"}}
```

注意：PowerShell 的 `curl` 可能是 `Invoke-WebRequest` 别名。如果输出不直观，可以用：

```powershell
Invoke-RestMethod http://localhost:8080/v1/users/1
```

---

## 十三、POST 请求测试

如果已经实现 `CreateUser`：

```powershell
Invoke-RestMethod `
  -Method Post `
  -Uri http://localhost:8080/v1/users `
  -ContentType 'application/json' `
  -Body '{"name":"Alice","email":"alice@example.com"}'
```

预期：

```json
{
  "user": {
    "id": "3",
    "name": "Alice",
    "email": "alice@example.com"
  }
}
```

如果还没实现 `CreateUser`，会返回 gRPC 默认未实现错误。

---

## 十四、常见错误排查

### 1. RegisterUserServiceHandlerFromEndpoint 未定义

说明没有生成 `.gw.go`，或者 import 的包不是生成文件所在包。

### 2. Gateway 启动成功，但 curl 返回 502 或连接失败

检查 gRPC server 是否启动在 `localhost:50051`。

### 3. HTTP 路由 404

检查 proto annotation 是否正确，是否重新生成 `.gw.go`。

### 4. PowerShell curl 输出奇怪

使用 `Invoke-RestMethod`，或者显式调用真正的 curl.exe：

```powershell
curl.exe http://localhost:8080/v1/users/1
```

---

## 十五、练习任务

1. 生成 `user.pb.gw.go`。
2. 创建 `cmd/gateway/main.go`。
3. 启动 gRPC server 和 HTTP gateway。
4. 用 curl 调用 GET /v1/users/1。
5. 用 Invoke-RestMethod 调用 POST /v1/users。
6. 故意关闭 gRPC server，观察 gateway 错误。

---

## 十六、完成标准

完成本节后，你应该能：

```text
安装 protoc-gen-grpc-gateway
生成 .gw.go 文件
编写 HTTP gateway main.go
启动 HTTP gateway
把 HTTP 请求转发到 gRPC server
使用 curl/Invoke-RestMethod 调用 HTTP API
排查 gateway 找不到 handler、路由 404、gRPC server 连接失败等问题
```

---

## 教程闭环检查

1. **完整操作步骤**：完成插件安装、代码生成、gateway main、server/gateway/curl 三终端验证。
2. **完整代码**：使用正文中的 `cmd/gateway/main.go`。
3. **运行命令**：执行 protoc、go mod tidy、go run server、go run gateway、curl。
4. **预期输出**：HTTP GET 能返回 JSON 用户信息。
5. **常见错误排查**：重点检查 `.gw.go`、gRPC 地址、路由 annotation、PowerShell curl。
6. **练习任务**：完成 GET、POST、关闭 gRPC server 三类实验。
7. **完成标准**：能独立启动一个 HTTP gateway 代理内部 gRPC 服务。