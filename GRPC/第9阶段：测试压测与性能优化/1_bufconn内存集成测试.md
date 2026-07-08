# 1. bufconn 内存集成测试

本节目标：使用 bufconn 在测试中启动内存 gRPC server，避免依赖真实端口。

bufconn 是 grpc-go 提供的测试工具，可以在内存里建立连接。它让你像真实 client 一样调用 server，又不用占用网络端口。

---

## 一、bufconn 适合测什么

- 生成 client 和 server 是否能正常协作。
- gRPC status code 是否正确。
- interceptor 是否生效。
- metadata 是否能传递。
- deadline 和 cancellation 是否符合预期。

它比普通业务单元测试更接近真实调用，比真实端口集成测试更稳定。

---

## 二、基本测试骨架

```go
const bufSize = 1024 * 1024
lis := bufconn.Listen(bufSize)

s := grpc.NewServer()
userv1.RegisterUserServiceServer(s, &userServer{})

go func() {
    if err := s.Serve(lis); err != nil {
        t.Logf("server exited: %v", err)
    }
}()
t.Cleanup(s.Stop)

dialer := func(context.Context, string) (net.Conn, error) {
    return lis.Dial()
}

ctx := context.Background()
conn, err := grpc.DialContext(ctx, "bufnet",
    grpc.WithContextDialer(dialer),
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)
if err != nil {
    t.Fatal(err)
}
t.Cleanup(func() { _ = conn.Close() })

client := userv1.NewUserServiceClient(conn)
```

---

## 三、断言错误码

```go
_, err = client.GetUser(ctx, &userv1.GetUserRequest{Id: -1})
if got := status.Code(err); got != codes.InvalidArgument {
    t.Fatalf("code=%s", got)
}
```

不要只比较错误字符串，字符串可能调整；code 才是稳定契约。

---

## 四、常见问题

- 测试结束不关闭 server：可能影响后续测试。
- 只测 service 层，不测 gRPC status：协议边界错误发现不了。
- 在测试中 sleep 等待：尽量用 context 和同步方式控制。
- 忘记注册拦截器：测试环境和真实环境行为不一致。

---

## 五、练习任务

1. 写一个 `TestGetUserOK`。
2. 写一个 `TestGetUserNotFound`。
3. 加入鉴权拦截器并测试无 token 失败。
4. 把测试初始化逻辑封装成 helper。

---

## 六、完成标准

- 测试不依赖真实端口。
- 能用生成 client 调用 server。
- 能断言 gRPC code。


---

## 七、完整操作步骤

本节目标：不用真实网络端口，在测试中启动一个内存 gRPC server，然后用真实生成的 client 调用它。

操作步骤：

1. 创建 bufconn listener。
2. 创建 gRPC server。
3. 注册 UserService。
4. 在 goroutine 中启动 server。
5. 使用自定义 dialer 连接 bufconn。
6. 创建真实 `UserServiceClient`。
7. 调用 `GetUser`。
8. 断言响应和错误码。
9. 测试结束时关闭 conn 和 server。

---

## 八、完整代码：测试 helper

创建 `user_server_test.go`：

```go
package main

import (
    "context"
    "net"
    "testing"

    userv1 "example.com/grpc-learning/proto/user/v1"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    "google.golang.org/grpc/test/bufconn"
)

const bufSize = 1024 * 1024

type testEnv struct {
    client userv1.UserServiceClient
    conn   *grpc.ClientConn
    server *grpc.Server
}

func newTestEnv(t *testing.T) *testEnv {
    t.Helper()

    lis := bufconn.Listen(bufSize)
    server := grpc.NewServer()
    userv1.RegisterUserServiceServer(server, newUserServer())

    go func() {
        if err := server.Serve(lis); err != nil {
            t.Logf("bufconn server stopped: %v", err)
        }
    }()

    dialer := func(context.Context, string) (net.Conn, error) {
        return lis.Dial()
    }

    conn, err := grpc.DialContext(
        context.Background(),
        "bufnet",
        grpc.WithContextDialer(dialer),
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        t.Fatalf("dial bufnet: %v", err)
    }

    env := &testEnv{
        client: userv1.NewUserServiceClient(conn),
        conn:   conn,
        server: server,
    }

    t.Cleanup(func() {
        _ = conn.Close()
        server.Stop()
        _ = lis.Close()
    })

    return env
}
```

这个 helper 会为每个测试创建一个独立内存 server。

---

## 九、完整代码：成功路径测试

```go
func TestGetUserOK(t *testing.T) {
    env := newTestEnv(t)

    resp, err := env.client.GetUser(context.Background(), &userv1.GetUserRequest{Id: 1})
    if err != nil {
        t.Fatalf("GetUser: %v", err)
    }

    user := resp.GetUser()
    if user.GetId() != 1 {
        t.Fatalf("id=%d", user.GetId())
    }
    if user.GetName() != "Tom" {
        t.Fatalf("name=%q", user.GetName())
    }
}
```

这里调用的是真实 gRPC client，不是直接调用 Go 函数。

---

## 十、完整代码：错误码测试

```go
func TestGetUserNotFound(t *testing.T) {
    env := newTestEnv(t)

    _, err := env.client.GetUser(context.Background(), &userv1.GetUserRequest{Id: 999})
    if err == nil {
        t.Fatal("expected error")
    }

    if got := status.Code(err); got != codes.NotFound {
        t.Fatalf("code=%s", got)
    }
}

func TestGetUserInvalidArgument(t *testing.T) {
    env := newTestEnv(t)

    _, err := env.client.GetUser(context.Background(), &userv1.GetUserRequest{Id: 0})
    if err == nil {
        t.Fatal("expected error")
    }

    if got := status.Code(err); got != codes.InvalidArgument {
        t.Fatalf("code=%s", got)
    }
}
```

需要 import：

```go
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)
```

---

## 十一、运行命令

运行所有测试：

```powershell
go test ./...
```

只运行 GetUser 测试：

```powershell
go test ./... -run TestGetUser -v
```

预期输出：

```text
=== RUN   TestGetUserOK
--- PASS: TestGetUserOK (0.00s)
=== RUN   TestGetUserNotFound
--- PASS: TestGetUserNotFound (0.00s)
=== RUN   TestGetUserInvalidArgument
--- PASS: TestGetUserInvalidArgument (0.00s)
PASS
```

---

## 十二、bufconn 的价值

bufconn 测到的是：

- proto 编解码。
- gRPC client/server 调用链。
- status code。
- interceptor。
- metadata。
- deadline。

它比直接调用：

```go
server.GetUser(ctx, req)
```

更接近真实 RPC，但又不需要监听真实 TCP 端口。

---

## 十三、常见错误排查

### 1. 测试卡住

检查 server 是否启动，dialer 是否返回 `lis.Dial()`。

### 2. code 是 Unknown

说明服务端可能返回了普通 error，而不是 status error。

### 3. 测试互相影响

如果多个测试共享全局 map，可能互相污染。建议每个测试创建新的 server 和数据。

### 4. 忘记 Cleanup

测试结束要关闭 conn、server、listener，避免资源泄露。

---

## 十四、练习任务

1. 写 `newTestEnv` helper。
2. 写 `TestGetUserOK`。
3. 写 `TestGetUserNotFound`。
4. 写 `TestGetUserInvalidArgument`。
5. 把 server 加上 logging interceptor，再确认测试仍通过。
6. 故意把 NotFound 改成普通 error，观察测试失败。

---

## 十五、完成标准

完成本节后，你应该能：

```text
使用 bufconn.Listen 创建内存 listener
在测试中启动 gRPC server
使用 grpc.WithContextDialer 连接 bufconn
用真实生成 client 调用 RPC
断言响应字段和 status code
用 t.Cleanup 释放资源
```

---

## 教程闭环检查

1. **完整操作步骤**：完成 helper、成功测试、错误码测试、Cleanup。
2. **完整代码**：使用正文中的 `newTestEnv`、`TestGetUserOK`、错误码测试。
3. **运行命令**：执行 `go test ./... -run TestGetUser -v`。
4. **预期输出**：三个测试 PASS。
5. **常见错误排查**：重点检查测试卡住、Unknown、共享状态、Cleanup。
6. **练习任务**：完成 interceptor 和普通 error 失败实验。
7. **完成标准**：能用 bufconn 写稳定的 gRPC 集成测试。