# 3. Metadata、Header 与 Trailer

本节目标：学会在 gRPC 调用中传递请求上下文和响应附加信息。

metadata 类似 HTTP Header，常用于传递 token、trace id、tenant id、language、client version 等信息。它不应该承载大业务数据，大业务数据应该放在 message 中。

---

## 一、核心直觉

- outgoing metadata：客户端发给服务端。
- incoming metadata：服务端从 context 中读取。
- header：服务端在响应开始阶段返回。
- trailer：服务端在响应结束阶段返回。
- metadata key 通常使用小写。

---

## 二、动手步骤

1. client 构造 metadata。
2. 使用 `metadata.NewOutgoingContext` 放入 context。
3. server 使用 `metadata.FromIncomingContext` 读取。
4. server 设置 header 或 trailer。

---

## 三、参考代码或命令

客户端：

```go
md := metadata.New(map[string]string{
    "authorization": "Bearer dev-token",
    "x-trace-id": "trace-001",
})
ctx := metadata.NewOutgoingContext(context.Background(), md)
```

服务端：

```go
md, ok := metadata.FromIncomingContext(ctx)
if ok {
    tokens := md.Get("authorization")
    traceIDs := md.Get("x-trace-id")
    _ = tokens
    _ = traceIDs
}
```

---

## 四、常见问题

- metadata key 使用大写导致行为不一致：建议统一小写。
- 把大 JSON 塞进 metadata：这会破坏边界，也不利于维护。
- 日志打印完整 token：必须脱敏。

---

## 五、练习任务

- client 传递 token 和 trace id。
- server 校验 token 是否存在。
- server 把处理耗时放到 trailer 中返回。

---

## 六、完成标准

- 能在 client 和 server 之间传递 metadata。
- 知道 metadata 和 message 的边界。
- 能避免敏感信息泄露。


---

## 七、完整操作步骤

本节做一个 metadata 实验：客户端通过 metadata 传 `authorization` 和 `x-trace-id`，服务端读取并校验 token，同时把 `x-server-version` 放到 header，把处理耗时放到 trailer。

操作顺序：

1. client 创建 outgoing metadata。
2. client 使用 `metadata.NewOutgoingContext` 包装 context。
3. server 使用 `metadata.FromIncomingContext` 读取 metadata。
4. server 校验 `authorization`。
5. server 设置 header。
6. server 设置 trailer。
7. client 接收 header/trailer。

---

## 八、完整客户端代码：发送 metadata

```go
func getUserWithMetadata(client userv1.UserServiceClient, id int64) {
    md := metadata.Pairs(
        "authorization", "Bearer dev-token",
        "x-trace-id", "trace-001",
    )

    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    ctx = metadata.NewOutgoingContext(ctx, md)

    var header metadata.MD
    var trailer metadata.MD

    resp, err := client.GetUser(
        ctx,
        &userv1.GetUserRequest{Id: id},
        grpc.Header(&header),
        grpc.Trailer(&trailer),
    )
    if err != nil {
        st, _ := status.FromError(err)
        log.Printf("GetUser failed: code=%s message=%s", st.Code(), st.Message())
        return
    }

    log.Printf("user=%s", resp.GetUser().GetName())
    log.Printf("header=%v", header)
    log.Printf("trailer=%v", trailer)
}
```

需要 import：

```go
import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/metadata"
)
```

---

## 九、完整服务端代码：读取 metadata

```go
func checkAuth(ctx context.Context) error {
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return status.Error(codes.Unauthenticated, "missing metadata")
    }

    tokens := md.Get("authorization")
    if len(tokens) == 0 {
        return status.Error(codes.Unauthenticated, "missing token")
    }

    if tokens[0] != "Bearer dev-token" {
        return status.Error(codes.Unauthenticated, "invalid token")
    }

    traceIDs := md.Get("x-trace-id")
    if len(traceIDs) > 0 {
        log.Printf("trace_id=%s", traceIDs[0])
    }

    return nil
}
```

在 `GetUser` 开头调用：

```go
if err := checkAuth(ctx); err != nil {
    return nil, err
}
```

---

## 十、完整服务端代码：设置 header 和 trailer

```go
func (s *userServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
    start := time.Now()

    if err := checkAuth(ctx); err != nil {
        return nil, err
    }

    header := metadata.Pairs("x-server-version", "v1.0.0")
    if err := grpc.SendHeader(ctx, header); err != nil {
        log.Printf("send header failed: %v", err)
    }

    defer func() {
        cost := time.Since(start).String()
        grpc.SetTrailer(ctx, metadata.Pairs("x-cost", cost))
    }()

    if req.Id <= 0 {
        return nil, status.Error(codes.InvalidArgument, "id must be positive")
    }

    user := s.users[req.Id]
    if user == nil {
        return nil, status.Error(codes.NotFound, "user not found")
    }

    return &userv1.GetUserResponse{User: user}, nil
}
```

header 更像响应开始时的附加信息，trailer 更像响应结束时的附加信息。

---

## 十一、运行命令

终端 1：

```powershell
go run ./cmd/server
```

终端 2：

```powershell
go run ./cmd/client
```

预期客户端输出：

```text
user=Tom
header=map[content-type:[application/grpc] x-server-version:[v1.0.0]]
trailer=map[x-cost:[1.2ms]]
```

预期服务端输出：

```text
trace_id=trace-001
```

---

## 十二、用 grpcurl 发送 metadata

```powershell
grpcurl -plaintext `
  -H "authorization: Bearer dev-token" `
  -H "x-trace-id: trace-from-grpcurl" `
  -d '{"id":1}' `
  localhost:50051 user.v1.UserService/GetUser
```

如果不传 token：

```powershell
grpcurl -plaintext -d '{"id":1}' localhost:50051 user.v1.UserService/GetUser
```

预期：

```text
ERROR:
  Code: Unauthenticated
  Message: missing token
```

---

## 十三、metadata 使用边界

适合放 metadata：

- authorization token。
- trace id。
- tenant id。
- request id。
- client version。
- language/locale。

不适合放 metadata：

- 大 JSON。
- 业务主体数据。
- 文件内容。
- 超大列表。
- 密码明文。

业务数据应该放在 request message 中，metadata 只传请求上下文和协议附加信息。

---

## 十四、常见错误排查

### 1. server 读取不到 metadata

client 必须用：

```go
ctx = metadata.NewOutgoingContext(ctx, md)
```

不要只是创建了 md 却没放进 ctx。

### 2. metadata key 大小写混乱

建议统一小写：

```text
authorization
x-trace-id
x-tenant-id
```

### 3. token 被完整打印到日志

不要打印完整 token。可以脱敏：

```text
Bearer dev-***
```

### 4. 把业务字段塞进 metadata

比如 `product_name`、`quantity` 应该放在 request message，不要放 metadata。

---

## 十五、练习任务

1. client 传 `authorization` 和 `x-trace-id`。
2. server 校验 token。
3. token 缺失时返回 `Unauthenticated`。
4. server 返回 `x-server-version` header。
5. server 返回 `x-cost` trailer。
6. 用 grpcurl 分别测试带 token 和不带 token。

---

## 十六、完成标准

完成本节后，你应该能：

```text
使用 metadata.NewOutgoingContext 发送 metadata
使用 metadata.FromIncomingContext 读取 metadata
用 metadata 传 token 和 trace id
区分业务数据和 metadata
使用 grpc.Header 接收 header
使用 grpc.Trailer 接收 trailer
知道 header 和 trailer 的差异
```

---

## 教程闭环检查

1. **完整操作步骤**：完成 metadata 发送、读取、校验、header/trailer 返回。
2. **完整代码**：使用正文中的 client、checkAuth、GetUser 示例。
3. **运行命令**：运行 server/client，并用 grpcurl 带 metadata 调用。
4. **预期输出**：能看到 user、header、trailer 和 trace id 日志。
5. **常见错误排查**：重点检查 metadata 是否放入 context、key 是否小写、token 是否脱敏。
6. **练习任务**：完成 token、trace id、header、trailer 实验。
7. **完成标准**：能解释 metadata/header/trailer 的职责边界。