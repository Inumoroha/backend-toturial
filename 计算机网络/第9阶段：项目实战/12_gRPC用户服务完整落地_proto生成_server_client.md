# 12. gRPC 用户服务完整落地：proto 生成、Server 与 Client

本节目标：完成一个最小 gRPC 用户服务，从 `.proto` 文件开始，生成 Go 代码，实现 server 和 client，并用命令验证调用结果。

gRPC 是 Go 后端服务间通信的常见选择。学习重点不是“背 HTTP/2 和 Protobuf 概念”，而是亲手完成一次从协议定义到服务调用的闭环。

---

## 一、最终目标

项目目录：

```text
grpc-user-lab/
  go.mod
  proto/user/v1/user.proto
  gen/user/v1/user.pb.go
  gen/user/v1/user_grpc.pb.go
  cmd/server/main.go
  cmd/client/main.go
```

服务定义：

```text
UserService.GetUser
UserService.CreateUser
```

你会练到：

- 安装 `protoc` 和 Go 插件。
- 编写 `.proto`。
- 生成 Go 代码。
- 实现 gRPC server。
- 编写 gRPC client。
- 处理 deadline 和错误码。

---

## 二、准备工具

确认 Go：

```powershell
go version
```

安装 Go 插件：

```powershell
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

确认插件在 PATH 中：

```powershell
protoc-gen-go --version
protoc-gen-go-grpc --version
```

如果 PowerShell 提示找不到命令，通常是 Go bin 目录没有加入 PATH。查看：

```powershell
go env GOPATH
```

插件一般在：

```text
%GOPATH%\bin
```

`protoc` 本体需要单独安装。Windows 可以通过包管理器或官网下载；Linux 可以使用系统包管理器。

---

## 三、创建项目

```powershell
mkdir grpc-user-lab
cd grpc-user-lab
go mod init example.com/grpc-user-lab
mkdir proto
mkdir proto\user
mkdir proto\user\v1
mkdir gen
mkdir cmd
mkdir cmd\server
mkdir cmd\client
```

安装依赖：

```powershell
go get google.golang.org/grpc
go get google.golang.org/protobuf
```

---

## 四、编写 proto 文件

创建：

```text
proto/user/v1/user.proto
```

写入：

```proto
syntax = "proto3";

package user.v1;

option go_package = "example.com/grpc-user-lab/gen/user/v1;userv1";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
}

message GetUserRequest {
  string id = 1;
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
```

逐项理解：

- `syntax = "proto3"` 表示使用 proto3 语法。
- `package user.v1` 是 Protobuf 层面的包名。
- `go_package` 决定生成后的 Go import path 和包名。
- 字段后面的 `= 1`、`= 2` 是字段编号，不是默认值。
- 字段编号一旦被线上使用，不要随便改。

---

## 五、生成 Go 代码

执行：

```powershell
protoc `
  --go_out=. `
  --go-grpc_out=. `
  proto/user/v1/user.proto
```

如果 `go_package` 写成上面的形式，生成文件可能位于：

```text
example.com/grpc-user-lab/gen/user/v1/
```

为了让输出更贴合当前项目，可以使用：

```powershell
protoc `
  --go_out=. --go_opt=module=example.com/grpc-user-lab `
  --go-grpc_out=. --go-grpc_opt=module=example.com/grpc-user-lab `
  proto/user/v1/user.proto
```

预期生成：

```text
gen/user/v1/user.pb.go
gen/user/v1/user_grpc.pb.go
```

检查：

```powershell
Get-ChildItem gen\user\v1
```

不要手改生成文件。需要修改接口时，改 `.proto` 后重新生成。

---

## 六、实现 gRPC Server

创建：

```text
cmd/server/main.go
```

写入：

```go
package main

import (
    "context"
    "log"
    "net"
    "strings"
    "sync"

    userv1 "example.com/grpc-user-lab/gen/user/v1"
    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

type userServer struct {
    userv1.UnimplementedUserServiceServer
    mu    sync.Mutex
    seq   int
    users map[string]*userv1.User
}

func newUserServer() *userServer {
    return &userServer{
        users: map[string]*userv1.User{
            "u_1": {Id: "u_1", Name: "Alice", Email: "alice@example.com"},
        },
    }
}

func (s *userServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
    id := strings.TrimSpace(req.GetId())
    if id == "" {
        return nil, status.Error(codes.InvalidArgument, "id is required")
    }

    s.mu.Lock()
    defer s.mu.Unlock()

    u := s.users[id]
    if u == nil {
        return nil, status.Error(codes.NotFound, "user not found")
    }

    return &userv1.GetUserResponse{User: u}, nil
}

func (s *userServer) CreateUser(ctx context.Context, req *userv1.CreateUserRequest) (*userv1.CreateUserResponse, error) {
    name := strings.TrimSpace(req.GetName())
    email := strings.TrimSpace(req.GetEmail())
    if name == "" || email == "" {
        return nil, status.Error(codes.InvalidArgument, "name and email are required")
    }

    s.mu.Lock()
    defer s.mu.Unlock()

    s.seq++
    id := "u_new_" + strconv.Itoa(s.seq)
    u := &userv1.User{
        Id:    id,
        Name:  name,
        Email: email,
    }
    s.users[id] = u

    return &userv1.CreateUserResponse{User: u}, nil
}
```

需要补 import：

```go
"strconv"
```

继续写 main 函数：

```go
func main() {
    ln, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatalf("listen failed: %v", err)
    }

    srv := grpc.NewServer()
    userv1.RegisterUserServiceServer(srv, newUserServer())

    log.Println("grpc user server listening on :50051")
    if err := srv.Serve(ln); err != nil {
        log.Fatalf("serve failed: %v", err)
    }
}
```

重点解释：

- `UnimplementedUserServiceServer` 用于保证未来新增方法时有默认实现。
- gRPC 错误不要随便返回普通字符串，推荐使用 `status.Error(codes.Xxx, "...")`。
- `InvalidArgument` 对应请求参数错误。
- `NotFound` 对应资源不存在。
- 这里用内存 map 模拟数据库，所以需要加锁。

---

## 七、实现 gRPC Client

创建：

```text
cmd/client/main.go
```

写入：

```go
package main

import (
    "context"
    "log"
    "time"

    userv1 "example.com/grpc-user-lab/gen/user/v1"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    "google.golang.org/grpc/status"
)

func main() {
    conn, err := grpc.Dial(
        "127.0.0.1:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        log.Fatalf("dial failed: %v", err)
    }
    defer conn.Close()

    c := userv1.NewUserServiceClient(conn)

    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    created, err := c.CreateUser(ctx, &userv1.CreateUserRequest{
        Name:  "Bob",
        Email: "bob@example.com",
    })
    if err != nil {
        log.Fatalf("CreateUser failed: %v", err)
    }
    log.Printf("created user: %+v", created.GetUser())

    got, err := c.GetUser(ctx, &userv1.GetUserRequest{
        Id: created.GetUser().GetId(),
    })
    if err != nil {
        log.Fatalf("GetUser failed: %v", err)
    }
    log.Printf("got user: %+v", got.GetUser())

    _, err = c.GetUser(ctx, &userv1.GetUserRequest{Id: "not-exists"})
    if err != nil {
        st, _ := status.FromError(err)
        log.Printf("expected error code=%s message=%s", st.Code(), st.Message())
    }
}
```

说明：

- 本地实验使用 `insecure.NewCredentials()`，表示不启用 TLS。
- 生产环境服务间通信通常要考虑 TLS、mTLS 或内网安全边界。
- 每次 RPC 都应该有 deadline，避免调用方无限等待。
- `status.FromError` 可以解析服务端返回的 gRPC 错误码。

---

## 八、运行验证

先整理依赖：

```powershell
go mod tidy
```

终端 1 启动 server：

```powershell
go run ./cmd/server
```

预期：

```text
grpc user server listening on :50051
```

终端 2 运行 client：

```powershell
go run ./cmd/client
```

预期日志类似：

```text
created user: id:"u_new_1" name:"Bob" email:"bob@example.com"
got user: id:"u_new_1" name:"Bob" email:"bob@example.com"
expected error code=NotFound message=user not found
```

这说明：

- 客户端成功调用了 `CreateUser`。
- 又用返回的 ID 调用了 `GetUser`。
- 不存在的 ID 被转换成了 gRPC `NotFound` 错误。

---

## 九、常见问题

### 1. 为什么生成文件路径不对？

通常是 `go_package` 和 `protoc` 参数没有配合好。

建议使用：

```powershell
--go_opt=module=example.com/grpc-user-lab
--go-grpc_opt=module=example.com/grpc-user-lab
```

这样生成路径会从模块根目录开始计算。

### 2. 为什么不要手改 `.pb.go` 文件？

因为它是生成物。

下一次执行 `protoc` 会覆盖你的修改。接口定义应该改 `.proto`，业务逻辑应该写在 server 实现里。

### 3. gRPC 和 HTTP JSON 怎么选？

常见经验：

```text
对外开放 API：HTTP JSON 更通用。
内部服务调用：gRPC 类型更明确，性能和代码生成体验更好。
浏览器直接访问：HTTP JSON 更自然。
强契约、多语言内部通信：gRPC 更合适。
```

### 4. gRPC 一定比 HTTP 快吗？

不能绝对说。

gRPC 使用 HTTP/2 和 Protobuf，通常在内部高频调用里有优势。但真实性能还取决于消息大小、连接复用、服务逻辑、部署网络和序列化成本。

### 5. deadline 放在哪里？

调用方必须设置 deadline。

服务端也应该尊重 `ctx.Done()`，在执行数据库查询、调用下游服务或长任务时及时退出。

---

## 十、扩展练习

继续完成：

1. 给 `CreateUser` 增加 email 格式校验。
2. 增加 `ListUsers` 服务端流式接口。
3. 给 server 增加 unary interceptor，记录方法名和耗时。
4. 使用 `grpcurl` 调用服务。
5. 为 gRPC 服务增加优雅关闭。

---

## 十一、本节达标标准

学完本节后，你应该能够做到：

- 编写包含 service 和 message 的 `.proto` 文件。
- 解释 `go_package` 的作用。
- 使用 `protoc-gen-go` 和 `protoc-gen-go-grpc` 生成 Go 代码。
- 实现 gRPC server。
- 注册 gRPC service。
- 编写 gRPC client。
- 为 RPC 调用设置 deadline。
- 使用 `codes.InvalidArgument`、`codes.NotFound` 表达业务错误。
- 说明 gRPC 和 HTTP JSON 的适用场景差异。

