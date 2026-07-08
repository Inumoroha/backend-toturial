# 3. HTTP 错误映射与响应格式

本节目标：理解 gRPC 错误如何映射为 HTTP 状态码，并设计统一响应格式。

内部 gRPC 调用方能理解 `codes.NotFound`，但 HTTP 调用方更熟悉 404、400、401、500。Gateway 需要把错误语义转换到 HTTP 世界。

---

## 一、常见映射关系

| gRPC code | HTTP status | 说明 |
| --- | --- | --- |
| InvalidArgument | 400 | 参数错误 |
| Unauthenticated | 401 | 未认证 |
| PermissionDenied | 403 | 无权限 |
| NotFound | 404 | 资源不存在 |
| AlreadyExists | 409 | 资源冲突 |
| Internal | 500 | 内部错误 |
| Unavailable | 503 | 服务不可用 |
| DeadlineExceeded | 504 | 超时 |

---

## 二、统一错误响应

建议对外响应稳定格式：

```json
{
  "code": "USER_NOT_FOUND",
  "message": "user not found",
  "traceId": "trace-001"
}
```

不要把数据库错误、堆栈、内部地址暴露给 HTTP 调用方。

---

## 三、错误来源

服务端应该返回标准 gRPC status：

```go
return nil, status.Error(codes.NotFound, "user not found")
```

Gateway 再把它映射到 HTTP 世界。这样内部 gRPC 调用和外部 HTTP 调用都能得到清晰错误。

---

## 四、常见问题

- gRPC 服务返回普通 error：Gateway 很难做精确映射。
- HTTP 响应格式每个接口不同：前端和第三方调用会痛苦。
- 把内部堆栈返回给外部：安全风险很高。
- 对外错误 message 写得太随意：会影响 API 使用体验。

---

## 五、练习任务

1. 分别触发 NotFound、InvalidArgument、Internal。
2. 观察 HTTP 状态码。
3. 设计统一错误响应 JSON。
4. 在日志中关联 trace id。

---

## 六、完成标准

- HTTP 错误状态和 gRPC code 对应清楚。
- 对外响应格式稳定。
- 内部错误不会泄露。


---

## 七、完整操作步骤

本节目标：观察 gRPC status code 如何通过 Gateway 映射成 HTTP status，并设计对外统一错误响应格式。

操作步骤：

1. 确保 gRPC server 对 id=0 返回 `InvalidArgument`。
2. 确保 id=999 返回 `NotFound`。
3. 启动 HTTP gateway。
4. 用 curl 调用成功、参数错误、资源不存在三种路径。
5. 观察 HTTP status code。
6. 了解默认错误响应格式。
7. 设计项目自己的错误响应格式。

---

## 八、服务端 gRPC 错误

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

Gateway 会根据 gRPC code 映射 HTTP status。

---

## 九、运行命令：观察 HTTP status

成功：

```powershell
curl.exe -i http://localhost:8080/v1/users/1
```

预期：

```text
HTTP/1.1 200 OK
```

参数错误：

```powershell
curl.exe -i http://localhost:8080/v1/users/0
```

预期：

```text
HTTP/1.1 400 Bad Request
```

资源不存在：

```powershell
curl.exe -i http://localhost:8080/v1/users/999
```

预期：

```text
HTTP/1.1 404 Not Found
```

---

## 十、常见映射表

| gRPC code | HTTP status | 说明 |
| --- | --- | --- |
| OK | 200 | 成功 |
| InvalidArgument | 400 | 参数错误 |
| Unauthenticated | 401 | 未认证 |
| PermissionDenied | 403 | 无权限 |
| NotFound | 404 | 资源不存在 |
| AlreadyExists | 409 | 冲突 |
| FailedPrecondition | 400 或 412 | 当前状态不满足操作 |
| ResourceExhausted | 429 | 资源耗尽或限流 |
| Internal | 500 | 内部错误 |
| Unavailable | 503 | 服务不可用 |
| DeadlineExceeded | 504 | 超时 |

实际映射以 gateway 版本和配置为准，但语义大致如此。

---

## 十一、默认错误响应

默认响应可能类似：

```json
{
  "code": 5,
  "message": "user not found",
  "details": []
}
```

这个格式能用，但对外 API 通常希望更稳定、更业务化：

```json
{
  "code": "USER_NOT_FOUND",
  "message": "user not found",
  "traceId": "trace-001"
}
```

---

## 十二、自定义错误处理思路

gRPC-Gateway 支持自定义 error handler。简化示意：

```go
mux := runtime.NewServeMux(
    runtime.WithErrorHandler(customHTTPErrorHandler),
)
```

自定义函数大致会拿到：

```go
func customHTTPErrorHandler(
    ctx context.Context,
    mux *runtime.ServeMux,
    marshaler runtime.Marshaler,
    w http.ResponseWriter,
    r *http.Request,
    err error,
) {
    st := status.Convert(err)
    httpStatus := runtime.HTTPStatusFromCode(st.Code())

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(httpStatus)

    _ = json.NewEncoder(w).Encode(map[string]any{
        "code":    st.Code().String(),
        "message": st.Message(),
    })
}
```

学习阶段先理解默认映射，项目阶段再统一封装错误响应。

---

## 十三、trace id 放进错误响应

如果 HTTP 请求带：

```text
X-Trace-Id: trace-001
```

Gateway 可以把它传给 gRPC metadata，也可以在错误响应中返回：

```json
{
  "code": "NOT_FOUND",
  "message": "user not found",
  "traceId": "trace-001"
}
```

这样调用方报错时，可以把 trace id 发给后端排查。

---

## 十四、常见错误排查

### 1. HTTP status 全是 500

检查 gRPC server 是否返回了标准 `status.Error`。如果返回普通 error，Gateway 可能无法准确映射。

### 2. 对外暴露内部错误

不要把 SQL、堆栈、内部地址返回给 HTTP 调用方。内部细节写日志。

### 3. 前端依赖 message 做逻辑判断

建议前端依赖稳定的业务 code，而不是自然语言 message。

### 4. gRPC code 和 HTTP status 语义不一致

例如用户不存在不应该映射成 500。先选对 gRPC code，HTTP 映射才会合理。

---

## 十五、练习任务

1. 用 curl.exe -i 调用成功路径，记录 HTTP 200。
2. 调用 `/v1/users/0`，记录 HTTP 400。
3. 调用 `/v1/users/999`，记录 HTTP 404。
4. 设计统一错误响应 JSON。
5. 思考 token 缺失应该返回 HTTP 401 还是 403。
6. 写一张 gRPC code 到 HTTP status 的项目映射表。

---

## 十六、完成标准

完成本节后，你应该能：

```text
解释 gRPC code 如何映射到 HTTP status
用 curl -i 观察 HTTP status
知道为什么服务端必须返回 status.Error
设计统一 HTTP 错误响应格式
避免内部错误泄露给外部调用方
理解 trace id 在错误排查中的作用
```

---

## 教程闭环检查

1. **完整操作步骤**：启动 server/gateway，分别触发成功、参数错误、资源不存在。
2. **完整代码**：使用正文中的 gRPC 错误和 customHTTPErrorHandler 思路。
3. **运行命令**：使用 curl.exe -i 查看 HTTP status。
4. **预期输出**：分别看到 200、400、404。
5. **常见错误排查**：重点检查普通 error、内部错误泄露、message 依赖、code/status 不一致。
6. **练习任务**：完成统一错误响应和映射表设计。
7. **完成标准**：能为 HTTP gateway 设计清晰的错误语义。