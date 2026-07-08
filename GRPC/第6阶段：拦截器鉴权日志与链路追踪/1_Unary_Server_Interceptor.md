# 1. Unary Server Interceptor

本节目标：掌握服务端 Unary 拦截器的签名、执行流程和典型用途。

Unary Server Interceptor 是最常用的拦截器。每次普通 RPC 进入服务端时，它都会先于业务 handler 执行，并且可以在 handler 返回后统一处理错误和日志。

---

## 一、签名拆解

```go
type UnaryServerInterceptor func(
    ctx context.Context,
    req any,
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (resp any, err error)
```

参数含义：

- `ctx`：请求上下文，可以读取 metadata、deadline、trace id。
- `req`：请求对象，类型是生成代码里的 request message。
- `info.FullMethod`：完整 RPC 方法名，例如 `/user.v1.UserService/GetUser`。
- `handler`：真正的业务方法入口。

最关键的一点是：如果你不调用 `handler(ctx, req)`，业务方法就不会执行。

---

## 二、日志拦截器

```go
func UnaryLoggingInterceptor(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
    start := time.Now()

    resp, err := handler(ctx, req)

    code := codes.OK
    if err != nil {
        code = status.Code(err)
    }

    log.Printf("method=%s code=%s duration=%s", info.FullMethod, code, time.Since(start))
    return resp, err
}
```

注册：

```go
s := grpc.NewServer(
    grpc.UnaryInterceptor(UnaryLoggingInterceptor),
)
```

---

## 三、推荐日志字段

一条可用的 RPC 日志至少应该包含：

```text
trace_id method code duration_ms peer user_id
```

学习阶段可以先使用标准库 `log`，真实项目建议使用结构化日志库，比如 zap、zerolog 或 slog。

---

## 四、常见问题

- 忘记调用 handler：业务方法不会执行。
- 调用 handler 多次：会造成重复业务操作。
- 直接打印完整请求体：可能泄露敏感信息，也可能导致日志过大。
- 不记录 code：排查错误时只能看到字符串，不容易统计。

---

## 五、练习任务

1. 记录 method、duration、code。
2. 给每个请求生成 request id。
3. 对敏感字段做脱敏后再打印。
4. 故意触发 NotFound，确认日志里能看到对应 code。

---

## 六、完成标准

- 拦截器能正常包住业务方法。
- 日志中能看到方法名、耗时和状态码。
- 知道 handler 是业务方法入口。


---

## 七、完整操作步骤

本节只聚焦一个目标：写出可运行的 Unary 日志拦截器，并确认每次 Unary RPC 都会打印方法名、状态码和耗时。

操作步骤：

1. 创建 `internal/middleware/logging.go`。
2. 写 `UnaryLoggingInterceptor`。
3. 在 server 中注册它。
4. 运行 server。
5. 用 client 或 grpcurl 调用 `GetUser`。
6. 观察服务端日志。
7. 分别触发 OK、NotFound、InvalidArgument，确认 code 正确。

---

## 八、完整代码：logging.go

```go
package middleware

import (
    "context"
    "log"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/metadata"
    "google.golang.org/grpc/status"
)

func UnaryLoggingInterceptor(
    ctx context.Context,
    req any,
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (any, error) {
    start := time.Now()

    resp, err := handler(ctx, req)

    code := codes.OK
    if err != nil {
        code = status.Code(err)
    }

    traceID := traceIDFromContext(ctx)
    log.Printf("method=%s code=%s duration=%s trace_id=%s",
        info.FullMethod,
        code,
        time.Since(start),
        traceID,
    )

    return resp, err
}

func traceIDFromContext(ctx context.Context) string {
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return ""
    }

    values := md.Get("x-trace-id")
    if len(values) == 0 {
        return ""
    }

    return values[0]
}
```

这个拦截器做了几件事：

```text
记录开始时间
调用业务 handler
根据 err 解析 gRPC code
从 metadata 中读取 trace id
打印 method/code/duration/trace_id
返回原始 resp 和 err
```

---

## 九、完整代码：server 注册

在 `cmd/server/main.go` 中：

```go
server := grpc.NewServer(
    grpc.UnaryInterceptor(middleware.UnaryLoggingInterceptor),
)

userv1.RegisterUserServiceServer(server, newUserServer())
reflection.Register(server)
```

需要 import：

```go
import "example.com/grpc-learning/internal/middleware"
```

如果你的 module 不是 `example.com/grpc-learning`，请改成自己的 module 路径。

---

## 十、运行命令

启动 server：

```powershell
go run ./cmd/server
```

使用 grpcurl 调用成功路径：

```powershell
grpcurl -plaintext `
  -H "x-trace-id: trace-ok" `
  -d '{"id":1}' `
  localhost:50051 user.v1.UserService/GetUser
```

调用 NotFound：

```powershell
grpcurl -plaintext `
  -H "x-trace-id: trace-not-found" `
  -d '{"id":999}' `
  localhost:50051 user.v1.UserService/GetUser
```

调用 InvalidArgument：

```powershell
grpcurl -plaintext `
  -H "x-trace-id: trace-invalid" `
  -d '{"id":0}' `
  localhost:50051 user.v1.UserService/GetUser
```

---

## 十一、预期输出

成功调用时，grpcurl 输出用户信息；服务端日志类似：

```text
method=/user.v1.UserService/GetUser code=OK duration=1.5ms trace_id=trace-ok
```

NotFound 时，grpcurl 输出：

```text
ERROR:
  Code: NotFound
  Message: user not found
```

服务端日志：

```text
method=/user.v1.UserService/GetUser code=NotFound duration=800µs trace_id=trace-not-found
```

InvalidArgument 时，服务端日志：

```text
method=/user.v1.UserService/GetUser code=InvalidArgument duration=600µs trace_id=trace-invalid
```

这说明拦截器对成功和失败都会执行。

---

## 十二、关键点：handler 必须只调用一次

拦截器里的核心是：

```go
resp, err := handler(ctx, req)
```

这行代码会真正调用业务方法。不要忘记，也不要调用两次。

错误示例：

```go
handler(ctx, req)
return handler(ctx, req)
```

这会让业务执行两次。如果是创建订单，可能产生两笔订单。

---

## 十三、常见错误排查

### 1. 日志没有打印

检查：

- server 是否使用了 `grpc.UnaryInterceptor`。
- 调用的是不是 Unary RPC。
- 是否启动了最新代码。

### 2. code 总是 OK

你可能没有从 err 中解析 code。正确写法：

```go
code := codes.OK
if err != nil {
    code = status.Code(err)
}
```

### 3. trace_id 为空

检查 client 或 grpcurl 是否传了：

```text
x-trace-id: trace-ok
```

### 4. 日志里打印完整请求体

不要在通用日志中直接打印整个 req。请求里可能包含密码、token、隐私数据。

---

## 十四、练习任务

1. 给日志增加 `peer` 地址。
2. 给日志增加 request id，如果没有 trace id 就生成一个。
3. 分别触发 OK、NotFound、InvalidArgument。
4. 故意让 handler 返回普通 error，观察 code 是什么。
5. 把日志格式改成一行 key=value，方便后续采集。

---

## 十五、完成标准

完成本节后，你应该能：

```text
写出 Unary Server Interceptor
解释 info.FullMethod 的含义
解释 handler(ctx, req) 的作用
记录 method/code/duration/trace_id
让成功和失败请求都被日志记录
避免重复调用 handler
```

---

## 教程闭环检查

1. **完整操作步骤**：创建 logging.go、注册 interceptor、运行三类请求。
2. **完整代码**：使用正文中的 UnaryLoggingInterceptor 和 server 注册代码。
3. **运行命令**：使用 grpcurl 触发 OK、NotFound、InvalidArgument。
4. **预期输出**：服务端日志应包含 method、code、duration、trace_id。
5. **常见错误排查**：重点检查 interceptor 是否注册、handler 是否调用、code 是否从 err 解析。
6. **练习任务**：完成 peer、trace id、普通 error、日志格式练习。
7. **完成标准**：能独立写出一个生产雏形的 Unary 日志拦截器。