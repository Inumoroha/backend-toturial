# 6. gRPC、HTTP/2、Protobuf 与服务间通信

本节目标：理解 gRPC 为什么适合服务间通信，以及它和 HTTP/2、Protobuf、context 的关系。

---

## 一、gRPC 的特点

gRPC 常用于内部服务通信。

特点：

- 基于 HTTP/2。
- 使用 Protobuf。
- 支持一元调用和流式调用。
- 生成强类型客户端和服务端代码。

---

## 二、proto 示例

```proto
syntax = "proto3";

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}

message GetUserRequest {
  int64 id = 1;
}

message GetUserResponse {
  int64 id = 1;
  string name = 2;
}
```

---

## 三、错误码

gRPC 常用 status code：

- `InvalidArgument`
- `NotFound`
- `AlreadyExists`
- `DeadlineExceeded`
- `Unavailable`
- `Internal`

Go 中：

```go
return nil, status.Error(codes.NotFound, "user not found")
```

---

## 四、超时和取消

客户端：

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()

resp, err := client.GetUser(ctx, req)
```

服务端应尊重 context 取消。

---

## 补充实践：gRPC 错误码和 HTTP 状态码的映射直觉

常见对应关系可以这样理解：

| gRPC code | 后端含义 | 类似 HTTP |
| --- | --- | --- |
| `InvalidArgument` | 参数错误 | 400 |
| `Unauthenticated` | 未认证 | 401 |
| `PermissionDenied` | 无权限 | 403 |
| `NotFound` | 资源不存在 | 404 |
| `AlreadyExists` | 唯一冲突 | 409 |
| `ResourceExhausted` | 限流或资源不足 | 429 |
| `DeadlineExceeded` | 调用超时 | 504 |
| `Unavailable` | 服务暂不可用 | 503 |
| `Internal` | 内部错误 | 500 |

不要所有错误都返回 `Internal`。错误码会影响调用方是否重试、是否报警、是否提示用户修改参数。

---

## 补充实践：服务端尊重 context

服务端处理耗时任务时，要检查上下文：

```go
func (s *Server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.GetUserResponse, error) {
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    default:
    }

    user, err := s.repo.GetUser(ctx, req.Id)
    if err != nil {
        return nil, err
    }
    return &pb.GetUserResponse{User: user}, nil
}
```

如果数据库库函数支持 context，就继续传下去。这样客户端取消或超时后，服务端不会继续做无意义的工作。

---

## 补充取舍：gRPC 不只是“更快的 HTTP”

选择 gRPC 的理由通常是：

```text
内部服务之间强契约。
多语言代码生成。
需要流式调用。
希望统一 deadline、metadata、status code。
高频结构化通信。
```

不选择 gRPC 的理由也要清楚：

```text
浏览器直接调用不方便。
调试门槛比 curl JSON 高。
需要维护 proto 兼容性。
部分网关、代理、观测工具要额外配置。
对外开放 API 时生态不如 HTTP JSON 通用。
```

所以不是“内部服务一定 gRPC”，而是看团队工具链、调用规模和契约管理能力。

---

## 五、常见问题

### 1. gRPC 是否适合浏览器直接调用？

普通浏览器直接调用不如 HTTP JSON 方便，通常需要 gRPC-Web 或网关转换。

### 2. Protobuf 是否一定比 JSON 好？

不一定。Protobuf 适合强类型内部通信，JSON 更适合开放接口和调试。

### 3. gRPC 超时应该放在哪里？

客户端必须设置 context timeout，服务端也要尊重 context 取消。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 解释 gRPC 和 HTTP/2 的关系。
- 说明 Protobuf 的作用。
- 使用 status code 表达错误。
- 在 gRPC 调用中使用 context timeout。
