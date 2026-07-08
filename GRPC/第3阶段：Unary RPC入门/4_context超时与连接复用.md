# 4. context 超时与连接复用

本节目标：理解 gRPC 调用中的超时控制和连接复用原则。

后端服务调用最怕无限等待。gRPC 调用必须认真设置 context 超时，并且要复用连接，避免每次请求都重新握手和建连。

---

## 一、核心直觉

- `context.WithTimeout` 表示最多等待多长时间。
- deadline 会传递给服务端，服务端可以通过 `ctx.Done()` 感知取消。
- gRPC 连接是相对重的对象，应该复用。
- client 可以是长期持有的依赖，而不是每个请求创建一次。

---

## 二、动手步骤

1. 给 client 的每次调用设置 1-3 秒超时。
2. 在 server 中模拟慢查询。
3. server 里监听 `ctx.Done()`。
4. 多次调用复用同一个 conn 和 client。

---

## 三、参考代码或命令

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

resp, err := client.GetUser(ctx, req)
```

服务端感知取消：

```go
select {
case <-time.After(5 * time.Second):
    return resp, nil
case <-ctx.Done():
    return nil, ctx.Err()
}
```

---

## 四、常见问题

- 认为服务端一定会自动停止：服务端业务代码要主动检查 context。
- 每次 HTTP 请求进来都 Dial 一次下游 gRPC：这会造成大量连接开销。
- timeout 设置过短或过长：要结合业务 SLO 和下游耗时。

---

## 五、练习任务

- 模拟服务端 sleep 5 秒，client 设置 1 秒超时。
- 在 server 日志中打印 context 取消。
- 把 client 连接封装成全局依赖或结构体字段。

---

## 六、完成标准

- 能解释 timeout 和 deadline。
- 能说明为什么连接要复用。
- server 能感知 client 取消。


---

## 七、为什么必须设置超时

后端系统中，最危险的调用不是“失败得很快”，而是“一直不返回”。如果一个 RPC 没有 deadline，下游服务卡住时，上游 goroutine、连接、内存都可能被拖住。

正确做法是每次调用都设置 context：

```go
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

resp, err := client.GetUser(ctx, req)
```

这表示：如果 3 秒内没有完成，客户端放弃等待。

---

## 八、服务端模拟慢请求

修改 server 的 `GetUser`：

```go
func (s *userServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
    if req.Id == 100 {
        select {
        case <-time.After(5 * time.Second):
        case <-ctx.Done():
            return nil, ctx.Err()
        }
    }

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

注意要 import：

```go
"time"
```

---

## 九、客户端制造超时

把 client 请求改成：

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()

resp, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: 100})
```

运行：

```powershell
go run ./cmd/client
```

预期看到类似：

```text
GetUser failed: code=DeadlineExceeded message=context deadline exceeded
```

这说明 client 不会无限等待。

---

## 十、服务端为什么要检查 ctx.Done

如果服务端写成：

```go
time.Sleep(5 * time.Second)
```

即使客户端 1 秒后已经超时，服务端仍然会继续 sleep 到 5 秒结束。这样会浪费资源。

更好的写法是：

```go
select {
case <-time.After(5 * time.Second):
    // 模拟操作完成
case <-ctx.Done():
    return nil, ctx.Err()
}
```

这样客户端取消后，服务端能尽快停止无意义工作。

---

## 十一、连接复用实验

命令行 demo 每次运行都会创建连接，这没问题。但你可以做一个小实验：在同一个进程里循环调用 5 次。

```go
for i := 0; i < 5; i++ {
    ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    resp, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: 1})
    cancel()
    if err != nil {
        log.Println(err)
        continue
    }
    log.Println(resp.GetUser().GetName())
}
```

注意：连接只创建一次，循环里只创建 context 和 request。

不要这样写：

```go
for i := 0; i < 5; i++ {
    conn, _ := grpc.Dial(...)
    client := userv1.NewUserServiceClient(conn)
    client.GetUser(...)
    conn.Close()
}
```

这种写法会重复建连。

---

## 十二、生产中的超时预算

假设一个 HTTP 请求总超时是 2 秒，内部要调用 user-service 和 product-service：

```text
HTTP request total timeout: 2s
order-service local work: 200ms
user-service call: 300ms
product-service call: 500ms
remaining buffer: 1000ms
```

不要给每个下游都设置 2 秒，否则整个链路可能远超用户等待时间。

---

## 十三、常见问题

### timeout 太短

如果设置：

```go
context.WithTimeout(context.Background(), time.Nanosecond)
```

大概率还没发出去就超时。这种设置只适合实验，不适合真实业务。

### timeout 太长

如果设置 1 分钟，下游故障时上游会长时间占着资源。真实项目应该根据接口 SLO 设置。

### defer cancel 忘记写

即使请求正常完成，也建议调用 cancel 释放 context 相关资源：

```go
defer cancel()
```

### 把 context 存到结构体里

不要把请求级 context 存到全局结构体。context 应该随着一次请求传递。

---

## 十四、练习任务

1. 给 GetUser 增加 id=100 的慢请求分支。
2. client 设置 1 秒超时调用 id=100。
3. 观察 DeadlineExceeded。
4. server 中打印 `ctx.Err()`。
5. 在同一个 client 连接上循环调用 5 次。

---

## 十五、完成标准

你应该能解释：

```text
context.WithTimeout 控制一次调用最长等待时间
deadline 会随 RPC 传给服务端
服务端应该检查 ctx.Done
连接应该复用
每次请求可以创建新的 context，但不应该创建新的连接
```
---

## 教程闭环检查

为了保证本节不是只停留在概念层面，学习时请按下面闭环完成：

1. **完整操作步骤**：先按正文顺序完成本节涉及的环境检查、文件创建、proto 编写、代码生成或运行验证。
2. **完整代码或命令**：本节如果涉及代码，请使用正文中的完整示例；如果是概念准备章节，请至少执行或记录正文给出的检查命令。
3. **运行命令**：把本节出现的关键命令实际运行一遍，例如 `go version`、`protoc --version`、`protoc ...`、`go run ...` 或 `grpcurl ...`。
4. **预期输出**：运行后对照正文中的预期输出；如果输出不同，先不要跳到下一节。
5. **常见错误排查**：遇到 PATH、go_package、端口占用、连接失败、生成文件缺失等问题时，优先按本节排错思路定位。
6. **练习任务**：完成正文中的练习，不只复制代码，要至少做一次参数或字段修改。
7. **完成标准**：能不看教程复述本节做了什么、为什么这样做、出错时从哪里查，才算真正完成。