# 2. 实现 gRPC 服务端

本节目标：实现并启动一个最小可用的 gRPC server。

服务端的工作是实现生成代码要求的接口，然后把实现注册到 `grpc.Server` 中。这里先使用内存数据，不接数据库，让注意力集中在 gRPC 本身。

---

## 一、核心直觉

- server 结构体需要嵌入 `UnimplementedUserServiceServer`，保证未来兼容新增方法。
- 方法签名由生成代码决定，不能自己随意改。
- `net.Listen` 负责监听端口。
- `grpc.NewServer` 创建 gRPC server。
- `RegisterUserServiceServer` 完成服务注册。

---

## 二、动手步骤

1. 创建 `cmd/server/main.go`。
2. 定义 `userServer` 结构体。
3. 实现 `GetUser` 方法。
4. 创建 listener。
5. 创建 gRPC server 并注册服务。
6. 调用 `Serve` 阻塞运行。

---

## 三、参考代码或命令

```go
type userServer struct {
    userv1.UnimplementedUserServiceServer
}

func (s *userServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
    return &userv1.GetUserResponse{
        User: &userv1.User{Id: req.Id, Name: "Tom", Email: "tom@example.com"},
    }, nil
}

func main() {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatal(err)
    }

    s := grpc.NewServer()
    userv1.RegisterUserServiceServer(s, &userServer{})

    log.Println("gRPC server listening on :50051")
    if err := s.Serve(lis); err != nil {
        log.Fatal(err)
    }
}
```

---

## 四、常见问题

- 忘记注册服务：server 虽然启动，但没有任何业务服务。
- 方法签名不匹配：要从生成接口复制签名，不要自己猜。
- 端口被占用：换端口或关闭已有进程。

---

## 五、练习任务

- 实现 `GetUser`。
- 当 `id == 0` 时先返回普通错误，下一阶段再改为 gRPC status。
- 启动服务并观察日志。

---

## 六、完成标准

- server 能成功监听 `:50051`。
- 无编译错误。
- 你知道注册服务和实现方法是两件事。


---

## 七、安装依赖

在工程根目录执行：

```powershell
go get google.golang.org/grpc
go get google.golang.org/grpc/reflection
go mod tidy
```

如果你只写最小 server，`reflection` 可以暂时不用；但建议开发阶段开启，方便后面用 grpcurl 调试。

---

## 八、完整服务端代码

创建 `cmd/server/main.go`：

```go
package main

import (
    "context"
    "log"
    "net"

    userv1 "example.com/grpc-learning/proto/user/v1"
    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/reflection"
    "google.golang.org/grpc/status"
)

type userServer struct {
    userv1.UnimplementedUserServiceServer
    users map[int64]*userv1.User
}

func newUserServer() *userServer {
    return &userServer{
        users: map[int64]*userv1.User{
            1: {Id: 1, Name: "Tom", Email: "tom@example.com"},
            2: {Id: 2, Name: "Jerry", Email: "jerry@example.com"},
        },
    }
}

func (s *userServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
    if req.Id <= 0 {
        return nil, status.Error(codes.InvalidArgument, "id must be positive")
    }

    user := s.users[req.Id]
    if user == nil {
        return nil, status.Error(codes.NotFound, "user not found")
    }

    return &userv1.GetUserResponse{User: user}, nil
}

func main() {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatalf("listen: %v", err)
    }

    server := grpc.NewServer()
    userv1.RegisterUserServiceServer(server, newUserServer())
    reflection.Register(server)

    log.Println("gRPC server listening on :50051")
    if err := server.Serve(lis); err != nil {
        log.Fatalf("serve: %v", err)
    }
}
```

如果你的 module 路径不是 `example.com/grpc-learning`，记得修改 import。

---

## 九、运行服务端

```powershell
go run ./cmd/server
```

预期输出：

```text
gRPC server listening on :50051
```

这个终端不要关闭。server 会一直阻塞运行，等待 client 调用。

---

## 十、用 grpcurl 验证服务

如果安装了 grpcurl，可以另开一个终端：

```powershell
grpcurl -plaintext localhost:50051 list
```

预期能看到：

```text
grpc.reflection.v1alpha.ServerReflection
user.v1.UserService
```

查看方法：

```powershell
grpcurl -plaintext localhost:50051 describe user.v1.UserService
```

调用：

```powershell
grpcurl -plaintext -d '{"id":1}' localhost:50051 user.v1.UserService/GetUser
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

## 十一、为什么要嵌入 UnimplementedUserServiceServer

```go
type userServer struct {
    userv1.UnimplementedUserServiceServer
}
```

这样做的作用是：当 proto 将来新增 RPC 方法时，旧 server 代码仍然可以编译，并对未实现方法返回默认错误。

如果不嵌入它，生成代码可能要求你实现所有方法，未来升级时更容易破坏编译。

---

## 十二、常见错误排查

### 端口被占用

错误：

```text
listen tcp :50051: bind: Only one usage of each socket address is normally permitted
```

解决：

- 关闭已经运行的 server。
- 或把端口改成 `:50052`。

### import 找不到 userv1

检查：

```powershell
go env GOMOD
Get-Content go.mod
```

确认 `go_package` 和 module 路径一致。

### grpcurl list 失败

如果错误提示 reflection 不可用，请确认 server 中有：

```go
reflection.Register(server)
```

---

## 十三、练习任务

1. 增加用户 id=3。
2. 调用 id=3，确认能返回。
3. 调用 id=999，观察 NotFound。
4. 调用 id=0，观察 InvalidArgument。
5. 关闭 reflection，再用 grpcurl list 观察失败。

---

## 十四、完成标准

你应该能独立完成：

```text
写 server main.go
注册 UserService
启动 :50051
用 grpcurl list 查看服务
用 grpcurl 调用 GetUser
根据错误信息排查端口和 import 问题
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