# 2. deadline、超时与取消传播

本节目标：掌握 gRPC 请求生命周期控制，避免调用无限等待。

分布式调用一定要有时间边界。没有超时的 RPC 调用会在下游故障时拖住上游资源，最终造成级联故障。

---

## 一、核心直觉

- timeout 是相对时间，例如 2 秒。
- deadline 是绝对截止时间。
- gRPC 会把 deadline 传给服务端。
- 服务端应该在耗时操作中检查 `ctx.Done()`。
- 上游取消后，下游应该尽快停止无意义工作。

---

## 二、动手步骤

1. client 设置 1 秒超时。
2. server 模拟 3 秒耗时操作。
3. client 观察 `DeadlineExceeded`。
4. server 通过 `ctx.Done()` 及时退出。

---

## 三、参考代码或命令

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()

_, err := client.GetUser(ctx, req)
```

服务端：

```go
select {
case <-time.After(3 * time.Second):
    return resp, nil
case <-ctx.Done():
    return nil, status.Error(codes.Canceled, "request canceled")
}
```

---

## 四、常见问题

- 服务端忽略 context：客户端已经走了，服务端还在占用资源。
- 每层调用都设置很长超时：链路总耗时会失控。
- 不区分 Canceled 和 DeadlineExceeded：一个是主动取消，一个是超时。

---

## 五、练习任务

- 模拟超时并打印错误码。
- 在服务端日志中打印取消原因。
- 设计一个订单创建链路的超时预算。

---

## 六、完成标准

- 能制造并识别超时。
- 服务端能响应取消。
- 知道超时预算应该逐层收敛。


---

## 七、完整操作步骤

本节目标是做一个可观察的超时实验：客户端设置 1 秒 deadline，服务端模拟 3 秒慢请求，最终客户端收到 `DeadlineExceeded`，服务端也能感知 `ctx.Done()`。

操作顺序：

1. 在 server 的 `GetUser` 中增加慢请求分支。
2. 服务端用 `select` 同时等待业务完成和 `ctx.Done()`。
3. 客户端设置 1 秒 timeout。
4. 客户端请求慢 id，例如 `id=100`。
5. 观察客户端错误码。
6. 观察服务端取消日志。

---

## 八、完整服务端代码

```go
func (s *userServer) GetUser(ctx context.Context, req *userv1.GetUserRequest) (*userv1.GetUserResponse, error) {
    if req.Id <= 0 {
        return nil, status.Error(codes.InvalidArgument, "id must be positive")
    }

    if req.Id == 100 {
        log.Println("simulate slow query for id=100")

        select {
        case <-time.After(3 * time.Second):
            log.Println("slow query finished")
        case <-ctx.Done():
            log.Printf("client canceled or deadline exceeded: %v", ctx.Err())
            return nil, ctx.Err()
        }
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
    "log"
    "time"
)
```

重点是 `select`。如果只写 `time.Sleep(3 * time.Second)`，服务端无法及时响应客户端取消。

---

## 九、准备测试数据

为了让 id=100 最终能查到用户，可以在内存 map 中加一条：

```go
users: map[int64]*userv1.User{
    1:   {Id: 1, Name: "Tom", Email: "tom@example.com"},
    100: {Id: 100, Name: "Slow User", Email: "slow@example.com"},
},
```

这样当 timeout 足够长时，id=100 可以成功；当 timeout 很短时，它会超时。

---

## 十、完整客户端代码

```go
func getUserWithTimeout(client userv1.UserServiceClient, id int64, timeout time.Duration) {
    ctx, cancel := context.WithTimeout(context.Background(), timeout)
    defer cancel()

    start := time.Now()
    resp, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: id})
    cost := time.Since(start)

    if err != nil {
        st, _ := status.FromError(err)
        log.Printf("GetUser failed: code=%s message=%s cost=%s", st.Code(), st.Message(), cost)
        return
    }

    log.Printf("GetUser success: user=%s cost=%s", resp.GetUser().GetName(), cost)
}
```

main 中测试两次：

```go
getUserWithTimeout(client, 100, time.Second)
getUserWithTimeout(client, 100, 5*time.Second)
```

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

预期客户端第一次输出：

```text
GetUser failed: code=DeadlineExceeded message=context deadline exceeded cost=1.0s
```

第二次输出：

```text
GetUser success: user=Slow User cost=3.0s
```

预期服务端第一次输出：

```text
simulate slow query for id=100
client canceled or deadline exceeded: context deadline exceeded
```

第二次输出：

```text
simulate slow query for id=100
slow query finished
```

---

## 十二、主动取消实验

除了 timeout，客户端也可以主动取消：

```go
ctx, cancel := context.WithCancel(context.Background())

go func() {
    time.Sleep(500 * time.Millisecond)
    cancel()
}()

_, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: 100})
```

预期 code 可能是：

```text
Canceled
```

这表示调用不是因为 deadline 到期，而是客户端主动取消。

---

## 十三、deadline 和 timeout 的区别

`timeout` 是相对时间：

```go
context.WithTimeout(context.Background(), time.Second)
```

意思是“从现在开始最多 1 秒”。

`deadline` 是绝对时间点：

```go
deadline := time.Now().Add(time.Second)
context.WithDeadline(context.Background(), deadline)
```

意思是“到这个具体时间点为止”。

实际项目里常用 `WithTimeout`，更直观。

---

## 十四、常见错误排查

### 1. 服务端没有感知取消

检查是否写了：

```go
case <-ctx.Done():
```

如果没有，客户端虽然超时了，服务端仍可能继续执行。

### 2. 超时设置太短

例如：

```go
context.WithTimeout(ctx, time.Nanosecond)
```

请求可能还没发出去就超时。实验可以这样做，业务不要这样写。

### 3. 超时设置太长

如果所有下游调用都设置 30 秒，故障时请求会堆积。真实项目要设计超时预算。

### 4. 忘记 defer cancel

正确写法：

```go
ctx, cancel := context.WithTimeout(...)
defer cancel()
```

这可以及时释放 context 相关资源。

---

## 十五、练习任务

1. 实现 id=100 慢请求。
2. client 分别设置 1 秒和 5 秒 timeout。
3. 观察 `DeadlineExceeded` 和成功返回。
4. 实现主动 cancel，观察 `Canceled`。
5. 设计一个创建订单链路的超时预算：总 2 秒，下游 user/product/order 各多少？

---

## 十六、完成标准

完成本节后，你应该能：

```text
使用 context.WithTimeout 设置调用超时
区分 DeadlineExceeded 和 Canceled
让服务端通过 ctx.Done 感知取消
解释 timeout 和 deadline 的区别
为跨服务调用设计超时预算
```

---

## 教程闭环检查

1. **完整操作步骤**：完成慢请求、短 timeout、长 timeout、主动 cancel 四个实验。
2. **完整代码**：使用正文中的 server 和 client 代码。
3. **运行命令**：运行 server/client 观察超时和成功输出。
4. **预期输出**：能看到 DeadlineExceeded、Canceled 或成功返回。
5. **常见错误排查**：重点检查服务端是否监听 `ctx.Done()`、timeout 是否合理、是否 `defer cancel()`。
6. **练习任务**：完成超时预算设计。
7. **完成标准**：能解释 deadline/cancellation 在 gRPC 调用链路中的传播。