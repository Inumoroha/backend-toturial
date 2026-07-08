# 3. 实现 gRPC 客户端

本节目标：使用生成的 client stub 调用 gRPC 服务。

客户端不需要关心服务端内部实现，它只依赖 proto 生成的 client。只要连接目标地址并创建 client，就可以像调用本地方法一样发起 RPC。

---

## 一、核心直觉

- client 通过连接对象创建，例如 `NewUserServiceClient(conn)`。
- 调用方法时必须传入 `context`。
- 连接要在程序结束时关闭。
- 学习阶段可以使用 insecure 连接，生产环境要使用 TLS。

---

## 二、动手步骤

1. 创建 `cmd/client/main.go`。
2. 连接 `localhost:50051`。
3. 创建 `UserServiceClient`。
4. 构造 `GetUserRequest`。
5. 调用 `GetUser` 并打印结果。

---

## 三、参考代码或命令

```go
conn, err := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
if err != nil {
    log.Fatal(err)
}
defer conn.Close()

client := userv1.NewUserServiceClient(conn)

ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

resp, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: 1})
if err != nil {
    log.Fatal(err)
}

log.Printf("user: %+v", resp.User)
```

---

## 四、常见问题

- server 没启动就运行 client：会连接失败。
- 地址写成 `http://localhost:50051`：gRPC Dial 通常写 host:port。
- 忘记超时：故障时体验很差。

---

## 五、练习任务

- 调用 `GetUser(1)` 并打印用户。
- 调用 `GetUser(0)` 观察错误。
- 修改超时时间为 1 毫秒，观察是否可能超时。

---

## 六、完成标准

- client 能成功拿到服务端返回。
- 能解释 client stub 的作用。
- 能使用 context 控制调用生命周期。


---

## 七、完整客户端代码

创建 `cmd/client/main.go`：

```go
package main

import (
    "context"
    "log"
    "time"

    userv1 "example.com/grpc-learning/proto/user/v1"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    "google.golang.org/grpc/status"
)

func main() {
    conn, err := grpc.Dial(
        "localhost:50051",
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        log.Fatalf("dial: %v", err)
    }
    defer conn.Close()

    client := userv1.NewUserServiceClient(conn)

    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    resp, err := client.GetUser(ctx, &userv1.GetUserRequest{Id: 1})
    if err != nil {
        st, _ := status.FromError(err)
        log.Fatalf("GetUser failed: code=%s message=%s", st.Code(), st.Message())
    }

    user := resp.GetUser()
    log.Printf("user id=%d name=%s email=%s", user.GetId(), user.GetName(), user.GetEmail())
}
```

说明：官方 gRPC Go 版本中也可以使用较新的连接 API。学习阶段如果使用 `grpc.Dial`，更容易和大量资料对应；如果你的依赖版本提示弃用，可以在后续再迁移到 `grpc.NewClient`。

---

## 八、运行客户端

先确保 server 正在运行，然后另开终端执行：

```powershell
go run ./cmd/client
```

预期输出：

```text
user id=1 name=Tom email=tom@example.com
```

如果看到连接失败，先检查 server 终端是否还在运行。

---

## 九、测试错误路径

把请求改成：

```go
&userv1.GetUserRequest{Id: 999}
```

再次运行：

```powershell
go run ./cmd/client
```

预期看到类似：

```text
GetUser failed: code=NotFound message=user not found
```

把 id 改成 0：

```go
&userv1.GetUserRequest{Id: 0}
```

预期：

```text
GetUser failed: code=InvalidArgument message=id must be positive
```

这说明 client 已经可以识别服务端返回的标准 gRPC code。

---

## 十、连接复用直觉

这个 demo 是命令行程序，运行一次就退出，所以只创建一次连接没有问题。

真实 HTTP 服务里，如果每个 HTTP 请求都这样做：

```go
conn, _ := grpc.Dial(...)
defer conn.Close()
```

就会有大量连接创建和关闭，性能很差。更合理的方式是：

```text
程序启动时创建 gRPC conn/client
多个请求复用同一个 client
程序退出时关闭连接
```

后续项目实战会采用这种方式。

---

## 十一、常见错误排查

### server 没启动

错误类似：

```text
connection refused
```

解决：先运行：

```powershell
go run ./cmd/server
```

### 地址写错

gRPC Dial 地址通常是：

```text
localhost:50051
```

不要写成：

```text
http://localhost:50051
```

### 忘记 insecure credentials

本地明文连接要写：

```go
grpc.WithTransportCredentials(insecure.NewCredentials())
```

生产环境不要用 insecure，要配置 TLS。

---

## 十二、练习任务

1. 把 id 改成 2，确认返回 Jerry。
2. 把 id 改成 999，确认 NotFound。
3. 把 timeout 改成 1 纳秒，观察错误。
4. 把 server 关闭后运行 client，观察连接错误。
5. 把 client 调用封装成 `getUser(client, id)` 函数。

---

## 十三、完成标准

你应该能独立完成：

```text
连接 gRPC server
创建 UserServiceClient
构造 GetUserRequest
设置 context timeout
调用 GetUser
解析 status code
打印响应
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