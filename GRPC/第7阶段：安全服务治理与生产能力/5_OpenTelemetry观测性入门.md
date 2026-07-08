# 5. OpenTelemetry 观测性入门

本节目标：理解 gRPC 服务需要哪些指标、日志和链路追踪信息。

服务上线后，最重要的问题从“能不能跑”变成“出问题时能不能看见”。观测性包括 logs、metrics、traces，gRPC 服务也需要这些能力。

---

## 一、三类观测数据

- Logs：记录离散事件，例如错误、鉴权失败、配置加载。
- Metrics：记录数值指标，例如 QPS、延迟、错误率。
- Traces：记录跨服务调用链路。

gRPC 最基础的维度：

```text
method code duration_ms peer trace_id
```

---

## 二、必须关注的指标

```text
grpc_server_requests_total{method,code}
grpc_server_duration_seconds{method}
grpc_client_requests_total{target,method,code}
grpc_client_duration_seconds{target,method}
```

你可以先用日志模拟这些信息，后续再接入 OpenTelemetry SDK 和 Prometheus。

---

## 三、trace id 传递

客户端通过 metadata 传递：

```go
md := metadata.Pairs("x-trace-id", traceID)
ctx := metadata.NewOutgoingContext(ctx, md)
```

服务端读取：

```go
md, _ := metadata.FromIncomingContext(ctx)
traceID := first(md.Get("x-trace-id"))
```

如果没有 trace id，入口服务应该生成一个，然后继续向下游传递。

---

## 四、常见问题

- 只有错误日志，没有延迟指标：性能问题很难定位。
- 每个服务自己生成 trace id 但不传递：链路断裂。
- 日志没有 method 和 code：排查 gRPC 问题不方便。
- 把 token、密码等敏感信息写入日志：安全风险高。

---

## 五、练习任务

1. 给日志加入 trace id。
2. 记录每个 RPC 的耗时。
3. 设计一张服务调用链路图。
4. 统计每个 method 的成功和失败次数。

---

## 六、完成标准

- 知道 gRPC 服务要观测哪些信息。
- trace id 能贯穿请求。
- 日志字段足够定位基本问题。


---

## 七、完整操作步骤

本节目标不是一次性接入完整 OpenTelemetry 后端，而是先建立可观测性的字段和埋点习惯。你要先能在日志中稳定看到 method、code、duration、trace_id，再理解这些字段如何进入 metrics 和 traces。

操作步骤：

1. 在 UnaryLoggingInterceptor 中记录 method、code、duration。
2. 从 metadata 读取或生成 trace id。
3. 日志中打印 trace id。
4. 设计 server metrics 名称。
5. 设计 client metrics 名称。
6. 理解 trace span 的边界。
7. 后续再接入 OpenTelemetry SDK 和 exporter。

---

## 八、完整代码：结构化日志字段

```go
func UnaryLoggingInterceptor(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
    start := time.Now()

    traceID := traceIDFromContext(ctx)
    if traceID == "" {
        traceID = newTraceID()
    }

    resp, err := handler(ctx, req)

    code := codes.OK
    if err != nil {
        code = status.Code(err)
    }

    log.Printf("trace_id=%s method=%s code=%s duration_ms=%d",
        traceID,
        info.FullMethod,
        code,
        time.Since(start).Milliseconds(),
    )

    return resp, err
}
```

生成 trace id 的简化版本：

```go
func newTraceID() string {
    return fmt.Sprintf("trace-%d", time.Now().UnixNano())
}
```

真实项目应使用 OpenTelemetry trace id，而不是时间戳字符串。

---

## 九、metadata 传递 trace id

client：

```go
md := metadata.Pairs("x-trace-id", "trace-001")
ctx := metadata.NewOutgoingContext(context.Background(), md)

resp, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: 1})
```

server 日志预期：

```text
trace_id=trace-001 method=/user.v1.UserService/GetUser code=OK duration_ms=2
```

如果没有传 trace id，server 生成一个：

```text
trace_id=trace-178xxx method=/user.v1.UserService/GetUser code=OK duration_ms=2
```

---

## 十、Metrics 设计

建议从这几个指标开始：

```text
grpc_server_requests_total{method,code}
grpc_server_request_duration_ms{method,code}
grpc_client_requests_total{target,method,code}
grpc_client_request_duration_ms{target,method,code}
```

这些指标能回答：

```text
哪个方法 QPS 最高？
哪个方法错误率最高？
哪个方法 P95/P99 延迟最高？
哪个下游最慢？
```

即使你暂时没有 Prometheus，也可以先在日志中把这些维度打出来。

---

## 十一、Trace 设计

一次跨服务调用可以想象成：

```text
HTTP Gateway Span
  -> OrderService.CreateOrder Span
      -> UserService.GetUser Span
      -> ProductService.DeductStock Span
```

每个 span 记录：

- span name。
- start/end time。
- status。
- error。
- attributes，例如 rpc.method、rpc.service、peer.address。

OpenTelemetry 最重要的价值是把多个服务的一次请求串起来。

---

## 十二、运行命令

启动 server：

```powershell
go run ./cmd/server
```

用 grpcurl 带 trace id 调用：

```powershell
grpcurl -plaintext `
  -H "authorization: Bearer dev-token" `
  -H "x-trace-id: trace-demo-001" `
  -d '{"id":1}' `
  localhost:50051 user.v1.UserService/GetUser
```

预期服务端日志：

```text
trace_id=trace-demo-001 method=/user.v1.UserService/GetUser code=OK duration_ms=2
```

触发错误：

```powershell
grpcurl -plaintext `
  -H "authorization: Bearer dev-token" `
  -H "x-trace-id: trace-demo-002" `
  -d '{"id":999}' `
  localhost:50051 user.v1.UserService/GetUser
```

预期日志：

```text
trace_id=trace-demo-002 method=/user.v1.UserService/GetUser code=NotFound duration_ms=1
```

---

## 十三、OpenTelemetry 接入位置

以后真正接入 OpenTelemetry 时，通常有两种方式：

### 方式一：使用生态拦截器

使用已有 gRPC OpenTelemetry instrumentation，自动创建 span 和 metrics。

### 方式二：自己在 interceptor 中埋点

在 Unary/Stream interceptor 中创建 span、记录属性、记录错误。

无论哪种方式，思想都是一样的：拦截器是 gRPC 观测性的最佳入口之一。

---

## 十四、常见错误排查

### 1. 日志没有 trace id

检查 client 是否传了 `x-trace-id`，server 是否从 metadata 读取。

### 2. 每个服务重新生成 trace id

这样链路会断。入口服务生成后，下游要继续传递。

### 3. 日志只有 message，没有 method/code

排查 gRPC 问题时必须有 method 和 code，否则很难统计。

### 4. 把 token 写进日志

观测性不是把所有东西都打出来。敏感字段必须脱敏或不记录。

---

## 十五、练习任务

1. 在 UnaryLoggingInterceptor 中打印 trace_id、method、code、duration_ms。
2. client 通过 metadata 传 `x-trace-id`。
3. 触发 OK 和 NotFound，观察日志差异。
4. 设计四个 metrics 名称。
5. 画出一次创建订单的 trace 树。
6. 写下哪些字段不能进入日志。

---

## 十六、完成标准

完成本节后，你应该能：

```text
解释 logs、metrics、traces 的区别
在 gRPC interceptor 中记录 method/code/duration/trace_id
通过 metadata 传递 trace id
设计基础 gRPC server/client metrics
理解跨服务 trace 的 span 结构
知道敏感信息不能进入日志
```

---

## 教程闭环检查

1. **完整操作步骤**：补齐日志字段、trace id 传递、OK/错误请求观察。
2. **完整代码**：使用正文中的 logging interceptor、newTraceID、metadata 示例。
3. **运行命令**：运行 server，用 grpcurl 带 trace id 调用 OK 和 NotFound。
4. **预期输出**：日志包含 trace_id、method、code、duration_ms。
5. **常见错误排查**：重点检查 trace id 断链、method/code 缺失、敏感信息泄露。
6. **练习任务**：完成 metrics 设计和 trace 树绘制。
7. **完成标准**：具备接入 OpenTelemetry 前的观测性建模能力。