# 1. status codes 错误模型

本节目标：学会使用 gRPC 标准状态码表达失败原因。

gRPC 错误不是简单字符串。服务端应该返回带状态码的错误，客户端应该根据状态码做处理。这样跨语言、跨服务调用时才能保持一致。

---

## 一、核心直觉

- `codes.InvalidArgument`：请求参数错误。
- `codes.NotFound`：资源不存在。
- `codes.AlreadyExists`：资源已存在。
- `codes.Unauthenticated`：未认证或 token 无效。
- `codes.PermissionDenied`：已认证但无权限。
- `codes.DeadlineExceeded`：超过 deadline。
- `codes.Internal`：服务内部错误，通常不要暴露细节。

---

## 二、动手步骤

1. 在服务端校验 `id > 0`。
2. 参数错误返回 `InvalidArgument`。
3. 用户不存在返回 `NotFound`。
4. client 解析错误码并分类处理。

---

## 三、参考代码或命令

```go
if req.Id <= 0 {
    return nil, status.Error(codes.InvalidArgument, "id must be positive")
}

user, ok := users[req.Id]
if !ok {
    return nil, status.Error(codes.NotFound, "user not found")
}
```

客户端：

```go
resp, err := client.GetUser(ctx, req)
if err != nil {
    st, _ := status.FromError(err)
    log.Printf("code=%s message=%s", st.Code(), st.Message())
    return
}
_ = resp
```

---

## 四、常见问题

- 把业务不存在返回 Internal：调用方无法区分服务故障和业务结果。
- 错误消息写得过于敏感：不要把 SQL、堆栈、密钥等返回给外部。
- client 只打印 error 字符串：应该解析 code。

---

## 五、练习任务

- 给 `CreateUser` 增加 `AlreadyExists`。
- 给鉴权失败增加 `Unauthenticated`。
- 写一个表格：业务场景 -> gRPC code。

---

## 六、完成标准

- 服务端错误码清晰。
- 客户端能按 code 分支处理。
- 能解释常用 code 的区别。


---

## 七、完整操作步骤

本节要把 `GetUser` 从“返回普通 error”改造成“返回标准 gRPC status code”。

操作顺序：

1. 在 server 中引入 `codes` 和 `status`。
2. 对请求参数做校验。
3. 用户不存在时返回 `NotFound`。
4. client 使用 `status.FromError` 解析错误。
5. 通过不同 id 触发不同错误。
6. 使用 grpcurl 观察错误输出。

---

## 八、完整服务端代码

修改 `cmd/server/main.go` 中的 `GetUser`：

```go
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
```

需要 import：

```go
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)
```

这两个包是 gRPC 错误处理的核心。

---

## 九、完整客户端代码

```go
func getUser(client userv1.UserServiceClient, id int64) {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    resp, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: id})
    if err != nil {
        st, ok := status.FromError(err)
        if !ok {
            log.Printf("not grpc error: %v", err)
            return
        }

        switch st.Code() {
        case codes.InvalidArgument:
            log.Printf("invalid request: %s", st.Message())
        case codes.NotFound:
            log.Printf("not found: %s", st.Message())
        case codes.Internal:
            log.Printf("server internal error")
        default:
            log.Printf("grpc error: code=%s message=%s", st.Code(), st.Message())
        }
        return
    }

    user := resp.GetUser()
    log.Printf("user id=%d name=%s email=%s", user.GetId(), user.GetName(), user.GetEmail())
}
```

main 中可以连续测试：

```go
getUser(client, 1)
getUser(client, 0)
getUser(client, 999)
```

---

## 十、运行命令

终端 1：

```powershell
go run ./cmd/server
```

终端 2：

```powershell
go run ./cmd/client
```

预期输出：

```text
user id=1 name=Tom email=tom@example.com
invalid request: id must be positive
not found: user not found
```

---

## 十一、使用 grpcurl 观察错误

成功：

```powershell
grpcurl -plaintext -d '{"id":1}' localhost:50051 user.v1.UserService/GetUser
```

参数错误：

```powershell
grpcurl -plaintext -d '{"id":0}' localhost:50051 user.v1.UserService/GetUser
```

预期：

```text
ERROR:
  Code: InvalidArgument
  Message: id must be positive
```

资源不存在：

```powershell
grpcurl -plaintext -d '{"id":999}' localhost:50051 user.v1.UserService/GetUser
```

预期：

```text
ERROR:
  Code: NotFound
  Message: user not found
```

---

## 十二、常用 code 选择表

| 场景 | 推荐 code | 说明 |
| --- | --- | --- |
| id <= 0 | InvalidArgument | 请求参数非法 |
| 用户不存在 | NotFound | 查询的资源不存在 |
| 邮箱已存在 | AlreadyExists | 创建时唯一冲突 |
| token 缺失 | Unauthenticated | 没有认证身份 |
| 角色无权限 | PermissionDenied | 有身份但没权限 |
| 下游服务不可用 | Unavailable | 临时不可用 |
| 超过 deadline | DeadlineExceeded | 请求超时 |
| 未预期 panic | Internal | 内部错误 |

注意 `Unauthenticated` 和 `PermissionDenied` 的区别：

```text
Unauthenticated：你是谁还没确认
PermissionDenied：知道你是谁，但你不能做这件事
```

---

## 十三、常见错误排查

### 1. client 看到 Unknown

通常是服务端返回了普通 error：

```go
return nil, fmt.Errorf("user not found")
```

改成：

```go
return nil, status.Error(codes.NotFound, "user not found")
```

### 2. 所有业务错误都用 Internal

`Internal` 应该留给服务内部异常，不应该表示用户不存在、参数错误、库存不足等业务状态。

### 3. 错误 message 暴露内部细节

不要返回：

```text
sql: select * from users failed: password=xxx
```

对外 message 应该安全、简洁。详细错误写日志。

### 4. client 依赖 message 判断错误

不要这样：

```go
if strings.Contains(err.Error(), "not found") {}
```

应该依赖：

```go
status.Code(err) == codes.NotFound
```

---

## 十四、练习任务

1. 给 `CreateUser` 设计 `AlreadyExists` 错误。
2. 给 `GetUser` 增加 id 小于 0 和 id 等于 0 的测试。
3. 用 grpcurl 分别触发 `InvalidArgument` 和 `NotFound`。
4. 写一个表格：你项目里的业务错误应该映射到哪个 code。
5. 思考库存不足应该用 `FailedPrecondition` 还是 `InvalidArgument`，写下理由。

---

## 十五、完成标准

完成本节后，你应该能：

```text
使用 status.Error 返回标准 gRPC 错误
使用 status.FromError 解析错误
根据业务场景选择合适 codes
区分 InvalidArgument、NotFound、Unauthenticated、PermissionDenied、Internal
避免用错误字符串做业务判断
```

---

## 教程闭环检查

1. **完整操作步骤**：改造 server 和 client 的错误处理。
2. **完整代码**：使用正文中的 `GetUser` 和 `getUser` 函数。
3. **运行命令**：运行 server/client，并用 grpcurl 触发错误。
4. **预期输出**：观察 InvalidArgument 和 NotFound 输出。
5. **常见错误排查**：重点检查 Unknown、Internal 滥用、message 泄露、字符串判断错误。
6. **练习任务**：完成 code 映射表和 grpcurl 错误实验。
7. **完成标准**：能稳定用 gRPC code 表达错误语义。