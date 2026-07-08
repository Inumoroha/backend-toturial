# 4. Stream Interceptor 与链式组合

本节目标：理解流式 RPC 的拦截器写法，以及多个拦截器如何组合。

流式 RPC 的拦截器签名和 Unary 不同，因为它拦截的是一条 stream。除了记录方法名和耗时，很多时候还要包装 stream，从 context 中注入信息。

---

## 一、Stream Interceptor 签名

```go
type StreamServerInterceptor func(
    srv any,
    ss grpc.ServerStream,
    info *grpc.StreamServerInfo,
    handler grpc.StreamHandler,
) error
```

参数说明：

- `srv`：服务实现对象。
- `ss`：服务端 stream。
- `info.FullMethod`：完整方法名。
- `handler`：真正的 stream handler。

---

## 二、日志拦截器

```go
func StreamLoggingInterceptor(srv any, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
    start := time.Now()
    err := handler(srv, ss)
    code := codes.OK
    if err != nil {
        code = status.Code(err)
    }
    log.Printf("stream_method=%s code=%s duration=%s", info.FullMethod, code, time.Since(start))
    return err
}
```

注册：

```go
s := grpc.NewServer(
    grpc.ChainUnaryInterceptor(RecoveryInterceptor, UnaryLoggingInterceptor, AuthInterceptor),
    grpc.ChainStreamInterceptor(StreamLoggingInterceptor),
)
```

---

## 三、什么时候需要包装 ServerStream

如果你想修改 stream 的 context，比如把鉴权后的 user id 写入 context，就需要包装 `grpc.ServerStream`。

```go
type wrappedStream struct {
    grpc.ServerStream
    ctx context.Context
}

func (w *wrappedStream) Context() context.Context {
    return w.ctx
}
```

然后把包装后的 stream 传给 handler。

---

## 四、常见问题

- 只给 Unary 加拦截器，忘了 Streaming：流式方法绕过鉴权或日志。
- 链式顺序混乱：例如 auth 在 recovery 外层时，auth panic 可能无法捕获。
- 在 stream interceptor 中假设有 req：流式消息不在拦截器参数里。
- 在 stream 中不处理 context：客户端断开后服务端可能继续工作。

---

## 五、练习任务

1. 给 `ListOrders` 增加 stream 日志。
2. 给 streaming 方法也加 token 校验。
3. 调整链顺序，观察日志差异。
4. 包装 ServerStream，把 trace id 写入 context。

---

## 六、完成标准

- Unary 和 Streaming 都有统一中间件。
- 能解释链式拦截器顺序。
- 知道流式拦截器和 Unary 的差别。


---

## 七、完整操作步骤

本节目标：给第 4 阶段的 streaming RPC 增加 Stream Interceptor，并理解链式组合的顺序。

操作步骤：

1. 创建 `internal/middleware/stream.go`。
2. 实现 `StreamLoggingInterceptor`。
3. 在 `grpc.NewServer` 中注册 `grpc.ChainStreamInterceptor`。
4. 运行第 4 阶段的 `ListOrders` 或 `ChatOrderStatus`。
5. 观察 stream 方法日志。
6. 学习如何包装 `grpc.ServerStream` 修改 context。
7. 为 stream 方法补 trace id 和鉴权思路。

---

## 八、完整代码：stream.go

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

func StreamLoggingInterceptor(
    srv any,
    ss grpc.ServerStream,
    info *grpc.StreamServerInfo,
    handler grpc.StreamHandler,
) error {
    start := time.Now()

    err := handler(srv, ss)

    code := codes.OK
    if err != nil {
        code = status.Code(err)
    }

    traceID := traceIDFromContext(ss.Context())
    log.Printf("stream_method=%s client_stream=%v server_stream=%v code=%s duration=%s trace_id=%s",
        info.FullMethod,
        info.IsClientStream,
        info.IsServerStream,
        code,
        time.Since(start),
        traceID,
    )

    return err
}
```

这段代码和 Unary Logging 很像，但参数不同：

```text
Unary: ctx + req + handler(ctx, req)
Stream: srv + ServerStream + handler(srv, stream)
```

---

## 九、server 注册 Stream Interceptor

如果是 OrderService：

```go
server := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        middleware.RecoveryInterceptor,
        middleware.UnaryLoggingInterceptor,
        middleware.AuthInterceptor,
    ),
    grpc.ChainStreamInterceptor(
        middleware.StreamLoggingInterceptor,
    ),
)

orderv1.RegisterOrderServiceServer(server, newOrderServer())
reflection.Register(server)
```

注意：Unary interceptor 不会自动作用于 streaming 方法。Streaming 方法必须注册 Stream interceptor。

---

## 十、运行命令

启动 order server：

```powershell
go run ./cmd/order-server
```

运行 Server Streaming client：

```powershell
go run ./cmd/order-client server-stream
```

或者如果没有命令行参数，就运行你第 4 阶段写好的 order client。

预期服务端日志：

```text
stream_method=/order.v1.OrderService/ListOrders client_stream=false server_stream=true code=OK duration=1.5s trace_id=trace-001
```

运行 Client Streaming：

```powershell
go run ./cmd/order-client client-stream
```

预期日志：

```text
stream_method=/order.v1.OrderService/CreateOrders client_stream=true server_stream=false code=OK duration=900ms trace_id=trace-001
```

运行双向流：

```powershell
go run ./cmd/order-client bidi-stream
```

预期日志：

```text
stream_method=/order.v1.OrderService/ChatOrderStatus client_stream=true server_stream=true code=OK duration=1.2s trace_id=trace-001
```

---

## 十一、包装 ServerStream 修改 context

Stream Interceptor 没有直接传入 `ctx` 参数。如果你想把鉴权结果、trace id 或 user id 写入 context，需要包装 `grpc.ServerStream`。

```go
type wrappedServerStream struct {
    grpc.ServerStream
    ctx context.Context
}

func (w *wrappedServerStream) Context() context.Context {
    return w.ctx
}
```

使用方式：

```go
func StreamAuthInterceptor(srv any, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) error {
    md, ok := metadata.FromIncomingContext(ss.Context())
    if !ok || len(md.Get("authorization")) == 0 {
        return status.Error(codes.Unauthenticated, "missing token")
    }

    ctx := context.WithValue(ss.Context(), userIDKey, int64(1001))
    wrapped := &wrappedServerStream{ServerStream: ss, ctx: ctx}

    return handler(srv, wrapped)
}
```

业务 stream handler 中调用 `stream.Context()` 时，拿到的就是包装后的 context。

---

## 十二、Stream Recovery 示例

可以写一个简化版 Stream Recovery：

```go
func StreamRecoveryInterceptor(srv any, ss grpc.ServerStream, info *grpc.StreamServerInfo, handler grpc.StreamHandler) (err error) {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("stream panic recovered method=%s panic=%v", info.FullMethod, r)
            err = status.Error(codes.Internal, "internal error")
        }
    }()

    return handler(srv, ss)
}
```

注册时：

```go
grpc.ChainStreamInterceptor(
    middleware.StreamRecoveryInterceptor,
    middleware.StreamLoggingInterceptor,
    middleware.StreamAuthInterceptor,
)
```

---

## 十三、链式顺序建议

Unary：

```text
Recovery -> Logging -> Auth -> Handler
```

Stream：

```text
StreamRecovery -> StreamLogging -> StreamAuth -> StreamHandler
```

为什么 Logging 放 Auth 前面？这样无 token 请求也能被记录。

为什么 Recovery 放最前面？这样后续拦截器和 handler panic 都尽量能捕获。

---

## 十四、常见错误排查

### 1. Streaming 方法没有日志

检查你是否注册了：

```go
grpc.ChainStreamInterceptor(...)
```

Unary interceptor 不会作用于 stream 方法。

### 2. 修改 context 后业务拿不到

检查是否包装了 `ServerStream` 并重写 `Context()` 方法。

### 3. Stream Auth 不生效

检查拦截器顺序和是否把 `StreamAuthInterceptor` 放进 `ChainStreamInterceptor`。

### 4. code 总是 Unknown

如果 handler 返回普通 error，`status.Code(err)` 可能是 Unknown。建议 stream handler 也返回 status error。

---

## 十五、完整代码：组合注册示例

```go
server := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        middleware.RecoveryInterceptor,
        middleware.UnaryLoggingInterceptor,
        middleware.AuthInterceptor,
    ),
    grpc.ChainStreamInterceptor(
        middleware.StreamRecoveryInterceptor,
        middleware.StreamLoggingInterceptor,
        middleware.StreamAuthInterceptor,
    ),
)
```

这个结构已经接近真实项目的基础形态。

---

## 十六、练习任务

1. 给 `ListOrders` 加 stream logging。
2. 给 `CreateOrders` 加 stream logging。
3. 给 `ChatOrderStatus` 加 stream logging。
4. 实现 `StreamAuthInterceptor`，无 token 返回 Unauthenticated。
5. 包装 ServerStream，把 user id 写入 context。
6. 实现 `StreamRecoveryInterceptor`，故意 panic 验证 Internal。

---

## 十七、完成标准

完成本节后，你应该能：

```text
写出 Stream Server Interceptor
解释 Stream interceptor 和 Unary interceptor 的签名差异
使用 ChainStreamInterceptor 注册多个 stream 拦截器
包装 grpc.ServerStream 修改 context
为 streaming RPC 添加 logging/auth/recovery
解释链式顺序对日志、鉴权、panic 捕获的影响
```

---

## 教程闭环检查

1. **完整操作步骤**：创建 stream.go，注册 StreamLogging/Auth/Recovery，运行三类 streaming RPC。
2. **完整代码**：使用正文中的 StreamLogging、wrappedServerStream、StreamAuth、StreamRecovery 示例。
3. **运行命令**：运行 order server/client 的 server-stream、client-stream、bidi-stream。
4. **预期输出**：服务端日志应显示 stream_method、client_stream、server_stream、code、duration、trace_id。
5. **常见错误排查**：重点检查 Stream interceptor 是否注册、context 是否包装、链式顺序是否正确。
6. **练习任务**：完成 stream logging、auth、recovery 和 context 注入。
7. **完成标准**：能给 streaming RPC 加上和 Unary 一致的工程化中间件能力。