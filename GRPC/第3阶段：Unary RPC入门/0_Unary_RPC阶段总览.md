# 0. Unary RPC 阶段总览

本节目标：从 proto 走到真正可运行的 gRPC server 和 client。

Unary RPC 是最常见的 gRPC 调用模式：一次请求，一次响应。绝大多数查询、创建、更新操作都可以先从 Unary 开始。

本阶段的目标是跑通完整链路：定义 proto，生成代码，实现服务端，编写客户端，设置超时，并理解连接复用。

---

## 一、核心直觉

- server 实现生成的 `UserServiceServer` 接口。
- client 使用生成的 `UserServiceClient` 调用远程方法。
- `context` 负责超时、取消和链路信息传递。
- gRPC 连接应该复用，不应该每次请求都重新创建。

---

## 二、动手步骤

1. 定义 `UserService.GetUser`。
2. 生成 Go 代码。
3. 实现 server 结构体和方法。
4. 启动 TCP 监听并注册服务。
5. 编写 client 调用服务。
6. 给调用加上超时。

---

## 三、参考代码或命令

```text
proto -> protoc -> generated code -> server implements interface -> client calls stub
```

---

## 四、常见问题

- 只实现方法但没有注册服务：server 启动了也无法响应 RPC。
- client 每次请求都新建连接：性能差，也容易耗尽连接资源。
- 忽略 context：没有超时的 RPC 在故障时可能一直等待。

---

## 五、练习任务

- 完成一个 `GetUser` demo。
- 用户存在返回用户；不存在先返回普通 error，下一阶段再改成标准 status code。
- 给 client 调用加 3 秒超时。

---

## 六、完成标准

- 能启动 server。
- 能运行 client 并得到结果。
- 能说清楚一次 Unary RPC 的完整流程。


---

## 七、本阶段最终要跑通什么

完成本阶段后，你应该能在两个终端里运行：

终端 1：

```powershell
go run ./cmd/server
```

看到：

```text
gRPC server listening on :50051
```

终端 2：

```powershell
go run ./cmd/client
```

看到：

```text
user id=1 name=Tom email=tom@example.com
```

这就是第一个完整 Unary RPC 闭环。

---

## 八、本阶段文件结构

```text
grpc-learning/
  go.mod
  proto/
    user/
      v1/
        user.proto
        user.pb.go
        user_grpc.pb.go
  cmd/
    server/
      main.go
    client/
      main.go
```

这个结构足够完成第一个 demo。后续再把业务逻辑拆到 `internal/user`。

---

## 九、调用链路图

```text
client main.go
  -> grpc.NewClient / grpc.Dial
  -> userv1.NewUserServiceClient
  -> client.GetUser(ctx, req)
  -> 网络传输
  -> server grpc.Server
  -> RegisterUserServiceServer 注册表
  -> userServer.GetUser
  -> 返回 GetUserResponse
```

你要重点理解：

- client 调的是生成代码里的 client stub。
- server 实现的是生成代码里的 server interface。
- 注册服务是把实现挂到 gRPC server 上。
- context 控制一次调用的生命周期。

---

## 十、学习顺序

1. 先确认 proto 和生成代码存在。
2. 写 server。
3. 写 client。
4. 先不加复杂错误处理，跑通成功路径。
5. 再加超时。
6. 再加 NotFound 这类错误。

不要一开始就把 interceptor、TLS、数据库都加进来，否则第一个闭环会变得太复杂。

---

## 十一、完成标准

你完成本阶段后，应该能解释：

```text
为什么 server 要嵌入 UnimplementedUserServiceServer
RegisterUserServiceServer 做了什么
NewUserServiceClient 做了什么
context.WithTimeout 为什么重要
为什么连接要复用
```


---

## 十二、Unary RPC 的适用场景

Unary RPC 是最像普通函数调用的一种模式，适合绝大多数“请求一次，返回一次”的业务。

常见场景：

```text
GetUser        查询用户
CreateUser     创建用户
GetOrder       查询订单
CreateOrder    创建订单
DeductStock    扣减库存
Login          登录
CheckPermission 权限检查
```

这些场景的共同点是：客户端一次性给出请求参数，服务端一次性返回结果。

如果服务端要持续返回很多条数据，比如导出 10 万条订单，就不适合把所有数据塞进一个 Unary response，后面应该用 Server Streaming。

---

## 十三、第一个 demo 为什么不接数据库

你可能会觉得用户服务应该接 PostgreSQL 或 MySQL。但第一个 gRPC demo 不建议接数据库，原因是：

- 数据库会引入连接池、SQL、迁移等额外复杂度。
- 初学阶段最重要的是理解 gRPC 调用链路。
- 内存 map 足够模拟业务数据。
- 等 gRPC 闭环跑通后，再接数据库会更清楚边界。

所以本阶段使用：

```go
map[int64]*userv1.User
```

这不是生产方案，只是学习方案。

---

## 十四、本阶段常见失败路径

你应该故意制造这些错误，观察现象：

1. server 不启动，直接运行 client。
2. server 端口改成 50052，client 仍连 50051。
3. proto 改了但忘记重新生成。
4. server 没有注册 UserService。
5. client 请求不存在的 id。
6. client 设置极短 timeout。

这些失败路径比成功路径更能帮助你理解 gRPC。

---

## 十五、本阶段完成后的复盘

写一段复盘，回答：

```text
1. proto 文件定义了什么？
2. 生成代码提供了什么？
3. server 实现了什么？
4. client 调用了什么？
5. context 控制了什么？
6. status code 表达了什么？
```

这段复盘可以写进你的 README。以后你学习 streaming、interceptor、gateway 时，都能回到这个最小闭环上理解。
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