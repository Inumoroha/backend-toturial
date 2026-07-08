# 2. 错误码、Metadata 与超时测试

本节目标：覆盖 gRPC 服务中最容易被忽略的边界行为。

成功路径测试只能证明服务在理想情况下可用。真实问题常出在非法参数、无权限、超时、metadata 缺失、客户端取消等边界上。

---

## 一、错误码测试

```go
_, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: -1})
if got := status.Code(err); got != codes.InvalidArgument {
    t.Fatalf("code=%s", got)
}
```

建议为每个核心接口覆盖：

- 成功。
- 参数错误。
- 资源不存在。
- 未认证。
- 内部错误兜底。

---

## 二、Metadata 测试

```go
md := metadata.Pairs("authorization", "Bearer test-token")
ctx := metadata.NewOutgoingContext(context.Background(), md)

_, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: 1})
```

至少测试三组：

- 不传 token。
- 传错误 token。
- 传正确 token。

---

## 三、超时测试

```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Millisecond)
defer cancel()

_, err := client.SlowMethod(ctx, req)
if got := status.Code(err); got != codes.DeadlineExceeded {
    t.Fatalf("code=%s", got)
}
```

超时测试不要设置得太极限，否则在不同机器上容易偶发失败。

---

## 四、常见问题

- 只比较 error 字符串：字符串可能变，但 code 应该稳定。
- timeout 设置太极限：测试容易偶发失败。
- 忘记测试无 metadata：鉴权拦截器可能空指针。
- 服务端不检查 context：客户端取消了，服务端还在跑。

---

## 五、练习任务

1. 为常用错误码各写一个测试。
2. 为 token 鉴权写三组测试。
3. 为 deadline 超时写测试。
4. 测试客户端主动 cancel。

---

## 六、完成标准

- 错误码测试稳定。
- metadata 边界被覆盖。
- 超时取消行为可验证。


---

## 七、完整操作步骤

本节目标：在 bufconn 测试中覆盖三类边界：错误码、metadata 鉴权、deadline 超时。

操作步骤：

1. 在测试 server 中注册 AuthInterceptor。
2. 写无 token 测试，断言 `Unauthenticated`。
3. 写正确 token 测试，断言成功。
4. 写错误 token 测试，断言 `Unauthenticated`。
5. 写 deadline 超时测试，断言 `DeadlineExceeded`。
6. 写主动 cancel 测试，断言 `Canceled` 或请求被取消。
7. 避免使用不稳定 sleep，给测试时间留余量。

---

## 八、完整代码：带拦截器的测试环境

```go
func newAuthTestEnv(t *testing.T) *testEnv {
    t.Helper()

    lis := bufconn.Listen(bufSize)
    server := grpc.NewServer(
        grpc.ChainUnaryInterceptor(
            middleware.AuthInterceptor,
        ),
    )
    userv1.RegisterUserServiceServer(server, newUserServer())

    go func() {
        if err := server.Serve(lis); err != nil {
            t.Logf("server stopped: %v", err)
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
        t.Fatalf("dial: %v", err)
    }

    t.Cleanup(func() {
        _ = conn.Close()
        server.Stop()
        _ = lis.Close()
    })

    return &testEnv{client: userv1.NewUserServiceClient(conn), conn: conn, server: server}
}
```

如果你的 `AuthInterceptor` 在 `internal/middleware` 包中，记得 import。

---

## 九、完整代码：metadata 鉴权测试

无 token：

```go
func TestGetUserMissingToken(t *testing.T) {
    env := newAuthTestEnv(t)

    _, err := env.client.GetUser(context.Background(), &userv1.GetUserRequest{Id: 1})
    if err == nil {
        t.Fatal("expected error")
    }

    if got := status.Code(err); got != codes.Unauthenticated {
        t.Fatalf("code=%s", got)
    }
}
```

正确 token：

```go
func TestGetUserWithToken(t *testing.T) {
    env := newAuthTestEnv(t)

    md := metadata.Pairs("authorization", "Bearer dev-token")
    ctx := metadata.NewOutgoingContext(context.Background(), md)

    resp, err := env.client.GetUser(ctx, &userv1.GetUserRequest{Id: 1})
    if err != nil {
        t.Fatalf("GetUser: %v", err)
    }

    if resp.GetUser().GetId() != 1 {
        t.Fatalf("id=%d", resp.GetUser().GetId())
    }
}
```

错误 token：

```go
func TestGetUserInvalidToken(t *testing.T) {
    env := newAuthTestEnv(t)

    md := metadata.Pairs("authorization", "Bearer wrong-token")
    ctx := metadata.NewOutgoingContext(context.Background(), md)

    _, err := env.client.GetUser(ctx, &userv1.GetUserRequest{Id: 1})
    if got := status.Code(err); got != codes.Unauthenticated {
        t.Fatalf("code=%s", got)
    }
}
```

---

## 十、完整代码：deadline 测试

假设 server 对 id=100 sleep 200ms：

```go
func TestGetUserDeadlineExceeded(t *testing.T) {
    env := newTestEnv(t)

    ctx, cancel := context.WithTimeout(context.Background(), 20*time.Millisecond)
    defer cancel()

    _, err := env.client.GetUser(ctx, &userv1.GetUserRequest{Id: 100})
    if err == nil {
        t.Fatal("expected error")
    }

    if got := status.Code(err); got != codes.DeadlineExceeded {
        t.Fatalf("code=%s", got)
    }
}
```

注意：测试超时不要设置得太极限，比如 1 纳秒，容易导致不稳定。20ms、50ms 这类值更容易复现。

---

## 十一、完整代码：主动取消测试

```go
func TestGetUserCanceled(t *testing.T) {
    env := newTestEnv(t)

    ctx, cancel := context.WithCancel(context.Background())
    cancel()

    _, err := env.client.GetUser(ctx, &userv1.GetUserRequest{Id: 1})
    if err == nil {
        t.Fatal("expected error")
    }

    if got := status.Code(err); got != codes.Canceled {
        t.Fatalf("code=%s", got)
    }
}
```

这测试的是客户端在调用前就取消 context 的情况。

---

## 十二、运行命令

```powershell
go test ./... -run 'TestGetUser.*Token|TestGetUserDeadline|TestGetUserCanceled' -v
```

如果 PowerShell 对单引号表达式处理不符合预期，可以直接运行：

```powershell
go test ./... -v
```

预期输出：

```text
--- PASS: TestGetUserMissingToken
--- PASS: TestGetUserWithToken
--- PASS: TestGetUserInvalidToken
--- PASS: TestGetUserDeadlineExceeded
--- PASS: TestGetUserCanceled
```

---

## 十三、常见错误排查

### 1. metadata 测试总是失败

检查测试 server 是否注册了 AuthInterceptor。如果你用的是不带拦截器的 `newTestEnv`，鉴权当然不会生效。

### 2. 正确 token 仍然 Unauthenticated

检查 metadata key 和 token 值是否完全一致。

### 3. deadline 测试偶发失败

测试时间太极限。把 server sleep 和 client timeout 拉开，例如 server 200ms，client 20ms。

### 4. 主动 cancel 返回其他错误

如果 cancel 时机不同，错误可能受调用阶段影响。调用前 cancel 最稳定。

---

## 十四、练习任务

1. 写无 token 测试。
2. 写错误 token 测试。
3. 写正确 token 测试。
4. 写 deadline exceeded 测试。
5. 写主动 cancel 测试。
6. 故意把 token key 改错，观察测试失败。
7. 把 server sleep 改小，观察 deadline 测试变化。

---

## 十五、完成标准

完成本节后，你应该能：

```text
在 bufconn 测试中注册 interceptor
通过 metadata.NewOutgoingContext 传 token
断言 Unauthenticated
断言 DeadlineExceeded
断言 Canceled
避免不稳定的时间测试
```

---

## 教程闭环检查

1. **完整操作步骤**：完成 metadata、deadline、cancel 三类测试。
2. **完整代码**：使用正文中的 newAuthTestEnv 和测试函数。
3. **运行命令**：执行 go test 运行相关测试。
4. **预期输出**：五个测试 PASS。
5. **常见错误排查**：重点检查拦截器注册、metadata key、时间余量、cancel 时机。
6. **练习任务**：完成 token key 和 sleep 调整实验。
7. **完成标准**：能自动化验证 gRPC 鉴权、超时和取消行为。