# 10. gRPC、grpcurl、拦截器与超时完整实验

本节目标：在 gRPC 基础上继续前进一步，学习如何用 `grpcurl` 调试服务，如何通过 unary interceptor 记录日志，如何在客户端设置 deadline，并观察超时错误。

第 9 阶段会做一个完整 gRPC 用户服务项目。本节更偏工程能力：调试、观测、拦截器和超时。

---

## 一、最终目标

项目目录：

```text
grpc-observe-lab/
  go.mod
  proto/echo/v1/echo.proto
  gen/echo/v1/echo.pb.go
  gen/echo/v1/echo_grpc.pb.go
  cmd/server/main.go
  cmd/client/main.go
```

服务能力：

```text
EchoService.Echo
EchoService.SlowEcho
```

你将完成：

- 开启 gRPC reflection，方便 `grpcurl` 查看服务。
- 编写 unary server interceptor，记录方法名、耗时和错误码。
- 客户端设置 deadline。
- 复现 `DeadlineExceeded`。

---

## 二、准备工具

安装 protoc 插件：

```powershell
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

安装 `grpcurl`：

```powershell
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest
```

确认：

```powershell
grpcurl -version
```

如果提示找不到命令，把 `go env GOPATH` 对应的 `bin` 目录加入 PATH。

---

## 三、创建项目

```powershell
mkdir grpc-observe-lab
cd grpc-observe-lab
go mod init example.com/grpc-observe-lab
mkdir proto
mkdir proto\echo
mkdir proto\echo\v1
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

## 四、编写 proto

创建：

```text
proto/echo/v1/echo.proto
```

写入：

```proto
syntax = "proto3";

package echo.v1;

option go_package = "example.com/grpc-observe-lab/gen/echo/v1;echov1";

service EchoService {
  rpc Echo(EchoRequest) returns (EchoResponse);
  rpc SlowEcho(EchoRequest) returns (EchoResponse);
}

message EchoRequest {
  string message = 1;
}

message EchoResponse {
  string message = 1;
}
```

生成代码：

```powershell
protoc `
  --go_out=. --go_opt=module=example.com/grpc-observe-lab `
  --go-grpc_out=. --go-grpc_opt=module=example.com/grpc-observe-lab `
  proto/echo/v1/echo.proto
```

检查：

```powershell
Get-ChildItem gen\echo\v1
```

---

## 五、实现 Server

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
    "time"

    echov1 "example.com/grpc-observe-lab/gen/echo/v1"
    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/reflection"
    "google.golang.org/grpc/status"
)

type echoServer struct {
    echov1.UnimplementedEchoServiceServer
}

func (s *echoServer) Echo(ctx context.Context, req *echov1.EchoRequest) (*echov1.EchoResponse, error) {
    msg := strings.TrimSpace(req.GetMessage())
    if msg == "" {
        return nil, status.Error(codes.InvalidArgument, "message is required")
    }
    return &echov1.EchoResponse{Message: msg}, nil
}

func (s *echoServer) SlowEcho(ctx context.Context, req *echov1.EchoRequest) (*echov1.EchoResponse, error) {
    select {
    case <-time.After(2 * time.Second):
        return &echov1.EchoResponse{Message: req.GetMessage()}, nil
    case <-ctx.Done():
        return nil, ctx.Err()
    }
}
```

`SlowEcho` 故意等待 2 秒。它同时监听 `ctx.Done()`，这样客户端超时后，服务端能及时停止等待。

---

## 六、实现 unary interceptor

继续在 server 文件中写：

```go
func loggingInterceptor(
    ctx context.Context,
    req any,
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (any, error) {
    start := time.Now()
    resp, err := handler(ctx, req)

    code := status.Code(err)
    log.Printf("grpc method=%s code=%s cost=%s",
        info.FullMethod,
        code,
        time.Since(start),
    )

    return resp, err
}
```

interceptor 的位置类似 HTTP 中间件：

```text
请求进入 -> interceptor -> 具体 RPC 方法 -> interceptor 收尾 -> 返回客户端
```

它适合做：

```text
日志。
指标。
链路追踪。
认证。
限流。
panic recover。
```

---

## 七、组合 Server main

继续写：

```go
func main() {
    ln, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatalf("listen failed: %v", err)
    }

    srv := grpc.NewServer(
        grpc.UnaryInterceptor(loggingInterceptor),
    )
    echov1.RegisterEchoServiceServer(srv, &echoServer{})

    reflection.Register(srv)

    log.Println("grpc echo server listening on :50051")
    if err := srv.Serve(ln); err != nil {
        log.Fatalf("serve failed: %v", err)
    }
}
```

`reflection.Register(srv)` 很适合开发和测试环境。它允许工具在不知道 proto 文件的情况下查询服务列表和方法定义。

生产环境是否开启 reflection，要看安全要求。

---

## 八、用 grpcurl 调试

启动服务：

```powershell
go run ./cmd/server
```

列出服务：

```powershell
grpcurl -plaintext localhost:50051 list
```

预期看到：

```text
echo.v1.EchoService
grpc.reflection.v1alpha.ServerReflection
```

查看服务方法：

```powershell
grpcurl -plaintext localhost:50051 list echo.v1.EchoService
```

调用 Echo：

```powershell
grpcurl -plaintext `
  -d '{\"message\":\"hello grpc\"}' `
  localhost:50051 echo.v1.EchoService/Echo
```

预期：

```json
{
  "message": "hello grpc"
}
```

调用参数错误：

```powershell
grpcurl -plaintext `
  -d '{\"message\":\"\"}' `
  localhost:50051 echo.v1.EchoService/Echo
```

预期看到 `InvalidArgument`。

---

## 九、实现 Client

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

    echov1 "example.com/grpc-observe-lab/gen/echo/v1"
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

    c := echov1.NewEchoServiceClient(conn)

    fastCtx, fastCancel := context.WithTimeout(context.Background(), time.Second)
    defer fastCancel()

    resp, err := c.Echo(fastCtx, &echov1.EchoRequest{Message: "hello"})
    if err != nil {
        log.Fatalf("Echo failed: %v", err)
    }
    log.Printf("Echo response: %s", resp.GetMessage())

    slowCtx, slowCancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
    defer slowCancel()

    _, err = c.SlowEcho(slowCtx, &echov1.EchoRequest{Message: "slow"})
    if err != nil {
        st, _ := status.FromError(err)
        log.Printf("SlowEcho expected error code=%s message=%s", st.Code(), st.Message())
    }
}
```

运行：

```powershell
go run ./cmd/client
```

预期：

```text
Echo response: hello
SlowEcho expected error code=DeadlineExceeded
```

服务端日志也会记录：

```text
grpc method=/echo.v1.EchoService/Echo code=OK cost=...
grpc method=/echo.v1.EchoService/SlowEcho code=DeadlineExceeded cost=...
```

---

## 十、观察 deadline 的意义

`SlowEcho` 需要 2 秒才返回，但客户端只愿意等 500ms：

```go
context.WithTimeout(context.Background(), 500*time.Millisecond)
```

所以客户端收到：

```text
DeadlineExceeded
```

如果服务端代码没有监听 `ctx.Done()`，服务端可能还会继续做无意义工作。好的服务端实现应该把上下文继续传给数据库、HTTP 下游、消息队列等耗时操作。

---

## 十一、常见问题

### 1. `grpcurl` 为什么要加 `-plaintext`？

因为本地实验没有启用 TLS。

如果服务启用了 TLS，就不能使用 `-plaintext`，需要配置证书或跳过校验参数。

### 2. 为什么 grpcurl 能不知道 proto 就调用？

因为服务端开启了 reflection。

`grpcurl` 可以通过 reflection 查询服务、方法、消息结构，再构造请求。

### 3. interceptor 和 HTTP middleware 一样吗？

思想相似，接口不同。

HTTP middleware 包装 `http.Handler`；gRPC interceptor 包装 RPC handler。它们都适合做横切逻辑。

### 4. 服务端返回 `ctx.Err()` 可以吗？

可以表达取消或超时，但业务错误更推荐使用 `status.Error(codes.Xxx, "...")`。

例如参数错误应该返回 `InvalidArgument`，资源不存在返回 `NotFound`。

---

## 十二、本节达标标准

学完本节后，你应该能够做到：

- 开启 gRPC reflection。
- 使用 `grpcurl list` 查看服务和方法。
- 使用 `grpcurl -d` 调用 unary RPC。
- 编写 unary server interceptor。
- 在 interceptor 中记录方法名、错误码和耗时。
- 在客户端为 RPC 设置 deadline。
- 复现并解释 `DeadlineExceeded`。
- 说明 interceptor 适合放哪些工程能力。

