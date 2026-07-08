# 3. Recovery 拦截器与 panic 处理

本节目标：用拦截器捕获 panic，避免单个请求导致服务进程崩溃。

业务代码可能因为空指针、数组越界等问题 panic。如果不处理，服务进程可能直接退出。Recovery interceptor 可以把 panic 转成 `codes.Internal`，同时记录日志。

---

## 一、核心原则

- recovery 应该放在拦截器链外层，尽量包住后续逻辑。
- 对外返回通用错误，不暴露堆栈。
- 对内日志要记录 panic 值和堆栈。
- recovery 不能替代正常错误处理，panic 应该是异常情况。

---

## 二、最小实现

```go
func RecoveryInterceptor(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp any, err error) {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("panic method=%s value=%v", info.FullMethod, r)
            err = status.Error(codes.Internal, "internal error")
        }
    }()

    return handler(ctx, req)
}
```

如果要记录堆栈，可以使用：

```go
buf := make([]byte, 64<<10)
n := runtime.Stack(buf, false)
log.Printf("panic stack: %s", string(buf[:n]))
```

---

## 三、链式顺序建议

```go
s := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        RecoveryInterceptor,
        UnaryLoggingInterceptor,
        AuthInterceptor,
    ),
)
```

这里 Recovery 放在最外层，尽量捕获后续拦截器和业务方法中的 panic。

---

## 四、常见问题

- 捕获 panic 后静默返回：没有日志就无法排查。
- 把 panic 原文返回给客户端：可能泄露内部信息。
- 用 panic 表达普通业务错误：业务错误应该返回 error。
- 只给 Unary 做 recovery：streaming 方法也需要保护。

---

## 五、练习任务

1. 在 `GetUser` 中临时写一个 panic。
2. 验证 client 收到 `codes.Internal`。
3. 验证 server 继续可用。
4. 给 Stream RPC 也加 recovery。

---

## 六、完成标准

- panic 不会导致进程退出。
- 日志能看到 panic 信息。
- 客户端只收到安全错误。


---

## 七、完整操作步骤

本节目标：写一个 Recovery interceptor，让业务 handler 中的 panic 不会导致整个 gRPC server 进程退出。

操作步骤：

1. 创建 `internal/middleware/recovery.go`。
2. 使用 `defer` + `recover()` 捕获 panic。
3. 记录 panic 值和堆栈。
4. 对客户端返回 `codes.Internal`。
5. 把 Recovery 放到拦截器链最外层。
6. 在 `GetUser` 中故意制造 panic。
7. 运行 client，确认收到 Internal，server 进程仍然存活。

---

## 八、完整代码：recovery.go

```go
package middleware

import (
    "context"
    "log"
    "runtime/debug"

    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

func RecoveryInterceptor(
    ctx context.Context,
    req any,
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (resp any, err error) {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("panic recovered method=%s panic=%v stack=%s", info.FullMethod, r, string(debug.Stack()))
            err = status.Error(codes.Internal, "internal error")
        }
    }()

    return handler(ctx, req)
}
```

这里使用命名返回值 `err error`，这样 defer 中可以把 panic 转成 gRPC error。

---

## 九、注册顺序

推荐：

```go
server := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        middleware.RecoveryInterceptor,
        middleware.UnaryLoggingInterceptor,
        middleware.AuthInterceptor,
    ),
)
```

Recovery 放最外层，尽量捕获后续 Logging、Auth、业务 handler 中的 panic。

如果你把 Recovery 放到最后：

```go
middleware.UnaryLoggingInterceptor,
middleware.AuthInterceptor,
middleware.RecoveryInterceptor,
```

那么 Logging/Auth 自己 panic 时，可能捕获不到。

---

## 十、制造 panic 实验

在 `GetUser` 中临时加：

```go
if req.Id == 500 {
    panic("mock panic for recovery test")
}
```

完整位置：

```go
func (s *userServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
    if req.Id == 500 {
        panic("mock panic for recovery test")
    }

    // other logic...
}
```

---

## 十一、运行命令

终端 1：

```powershell
go run ./cmd/server
```

终端 2：

```powershell
grpcurl -plaintext `
  -H "authorization: Bearer dev-token" `
  -d '{"id":500}' `
  localhost:50051 user.v1.UserService/GetUser
```

预期客户端输出：

```text
ERROR:
  Code: Internal
  Message: internal error
```

预期服务端日志包含：

```text
panic recovered method=/user.v1.UserService/GetUser panic=mock panic for recovery test
```

并且 server 进程仍然继续运行。你可以再调用 id=1 验证：

```powershell
grpcurl -plaintext `
  -H "authorization: Bearer dev-token" `
  -d '{"id":1}' `
  localhost:50051 user.v1.UserService/GetUser
```

应该能正常返回用户。

---

## 十二、为什么不能把 panic 原文返回给客户端

不要这样：

```go
err = status.Error(codes.Internal, fmt.Sprintf("panic: %v", r))
```

panic 里可能包含：

- SQL。
- 文件路径。
- 内部服务地址。
- 敏感参数。
- 堆栈信息。

这些应该写到服务端日志，而不是返回给调用方。

---

## 十三、常见错误排查

### 1. panic 后服务仍然退出

检查 Recovery 是否注册到了 `grpc.NewServer`，并且是否在链条外层。

### 2. client 看到 Unknown 而不是 Internal

可能是 panic 没有被 recovery 捕获，或者 recovery 返回了普通 error。应该返回：

```go
status.Error(codes.Internal, "internal error")
```

### 3. 没有堆栈日志

建议记录：

```go
debug.Stack()
```

否则排查 panic 很困难。

### 4. 用 panic 表达业务错误

不要用 panic 表示用户不存在、库存不足、参数错误。这些应该返回普通 error 或 status error。

---

## 十四、Stream Recovery 预告

Unary Recovery 只能保护 Unary RPC。Streaming 方法需要 Stream Interceptor 或在 handler 内部 recover。

第 6 阶段最后一节会介绍 Stream Interceptor 的写法。真实项目中 Unary 和 Stream 都要有 recovery。

---

## 十五、练习任务

1. 实现 `RecoveryInterceptor`。
2. 把它放到链式拦截器最外层。
3. 在 `GetUser` 中对 id=500 panic。
4. 用 grpcurl 调用 id=500，确认返回 Internal。
5. 再调用 id=1，确认服务仍然可用。
6. 查看服务端日志中是否包含 stack。

---

## 十六、完成标准

完成本节后，你应该能：

```text
使用 defer recover 捕获 panic
把 panic 转成 codes.Internal
记录 panic 值和堆栈
避免把内部细节返回给客户端
理解 Recovery interceptor 的链式顺序
区分 panic 和普通业务错误
```

---

## 教程闭环检查

1. **完整操作步骤**：创建 recovery.go、注册 Recovery、制造 panic、验证服务不退出。
2. **完整代码**：使用正文中的 RecoveryInterceptor 和 id=500 panic 实验。
3. **运行命令**：运行 server，用 grpcurl 调用 id=500 和 id=1。
4. **预期输出**：id=500 返回 Internal，id=1 仍正常返回。
5. **常见错误排查**：重点检查注册顺序、Internal 返回、堆栈日志和 panic 滥用。
6. **练习任务**：完成 panic 恢复和服务存活验证。
7. **完成标准**：能用 Recovery interceptor 保护 Unary RPC。