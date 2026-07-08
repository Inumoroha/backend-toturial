# 1. 理解 RPC 与 gRPC 解决的问题

本节目标：理解为什么后端服务需要 RPC，以及 gRPC 在服务间通信中的位置。

在单体应用里，函数调用发生在同一个进程中；在微服务或分布式系统里，订单服务要查询用户服务，调用就跨进程、跨机器了。RPC 要解决的就是这个跨进程调用问题。

如果直接使用 HTTP/JSON，你需要手写 URL、方法、Header、JSON 编解码和错误处理。gRPC 则用 `.proto` 先定义契约，再生成强类型 client 和 server 代码。

---

## 一、核心直觉

- RPC 不是某一种协议，而是一种通信方式：让远程调用看起来像本地方法调用。
- gRPC 是 RPC 框架，Protobuf 是默认的接口定义和序列化方式。
- HTTP/2 给 gRPC 提供多路复用、流式传输、Header/Trailer 等能力。
- gRPC 的强项是内部服务调用、低延迟通信、强契约和跨语言。
- REST/HTTP 的强项是开放性、浏览器友好、调试简单、生态普遍。

---

## 二、动手步骤

1. 先想一个业务场景：订单服务调用用户服务获取用户信息。
2. 用 HTTP/JSON 思考一次调用要写哪些内容：URL、Method、JSON、状态码、鉴权。
3. 再用 gRPC 思考：只要拿到生成的 client，调用 `GetUser(ctx, req)`。
4. 对比两种方式的开发体验和维护成本。

---

## 三、参考代码或命令

HTTP/JSON 思维：

```text
GET http://user-service/users/1001
Header: Authorization: Bearer xxx
Response: {"id":1001,"name":"Tom"}
```

gRPC 思维：

```go
resp, err := userClient.GetUser(ctx, &userv1.GetUserRequest{Id: 1001})
```

---

## 四、常见问题

- 以为 gRPC 只能用于微服务：其实本地工具、边缘服务、内部 SDK 都可能使用它。
- 以为用了 gRPC 就不需要接口设计：gRPC 更依赖契约设计，proto 设计不好，后续演进会痛苦。
- 以为 gRPC 一定比 HTTP 快：性能取决于场景、连接复用、消息大小、序列化和服务实现。

---

## 五、练习任务

- 写出 3 个适合 gRPC 的场景。
- 写出 3 个更适合 HTTP/JSON 的场景。
- 用自己的话解释“强契约”是什么意思。

---

## 六、完成标准

- 能说清楚 gRPC 的适用边界。
- 能解释为什么服务间调用经常选择 gRPC。
- 不会把 gRPC 简单理解成“更快的 HTTP”。


---

## 七、从一个具体场景理解 RPC

假设你正在做一个订单系统，创建订单时需要拿到用户信息。最直接的 HTTP 写法可能是：

```go
resp, err := http.Get("http://user-service/users/1001")
if err != nil {
    return err
}
defer resp.Body.Close()

if resp.StatusCode != http.StatusOK {
    return fmt.Errorf("user service status: %d", resp.StatusCode)
}

var user User
if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
    return err
}
```

这段代码没有错，但你要自己处理很多细节：

- URL 怎么拼。
- HTTP method 用什么。
- JSON 字段是否匹配。
- 状态码如何解释。
- 超时如何设置。
- 鉴权 Header 如何传。
- 返回结构变更时如何发现。

RPC 的思路是先定义契约，再生成调用代码。调用方看到的是：

```go
user, err := userClient.GetUser(ctx, &userv1.GetUserRequest{Id: 1001})
```

这个调用仍然是远程调用，不是真的本地函数，但它有更强的类型约束和更清晰的接口边界。

---

## 八、RPC 调用背后发生了什么

一次 gRPC Unary 调用大致可以拆成：

```text
client 构造 request message
-> client stub 把 message 序列化
-> 通过 HTTP/2 发送到 server
-> server 根据 service/method 找到 handler
-> handler 执行业务逻辑
-> server 返回 response message 或 status error
-> client stub 反序列化 response
-> 调用方拿到 Go 结构体
```

你不需要一开始就掌握所有底层细节，但要知道 gRPC 并不是魔法。它只是把网络通信、序列化、服务路由、错误模型等能力封装成了一套规范。

---

## 九、gRPC 与 REST 的取舍

### 更适合 gRPC 的场景

- 内部微服务之间频繁调用。
- 调用双方都由同一团队或可信团队维护。
- 对类型安全和接口契约要求高。
- 需要 streaming，例如实时进度、状态推送、文件分片。
- 多语言服务之间需要统一接口定义。

### 更适合 HTTP/JSON 的场景

- 浏览器直接调用。
- 对外开放 API。
- 第三方开发者接入。
- 希望使用 curl、浏览器、Postman 直接调试。
- 接口数据结构简单，性能不是主要矛盾。

真实项目里经常是混合架构：

```text
外部用户 -> HTTP/JSON Gateway -> 内部 gRPC 服务
```

这也是后面要学习 gRPC-Gateway 的原因。

---

## 十、动手练习：手写一次对比

请创建一个 `rpc-thinking.md`，写下下面内容：

```markdown
# RPC 思维练习

## 业务场景
订单服务创建订单前需要查询用户。

## HTTP/JSON 设计
- Method:
- URL:
- Request Header:
- Response JSON:
- 错误状态码:

## gRPC 设计
- Service:
- RPC Method:
- Request Message:
- Response Message:
- 错误 code:

## 我的理解
RPC 适合这里的原因是：...
```

这不是形式主义。你写完之后，会更清楚接口契约到底应该放在哪里。

---

## 十一、常见错误理解

### “gRPC 就是 HTTP/2”

不准确。gRPC 通常基于 HTTP/2，但 gRPC 还包括：

- IDL 设计。
- Protobuf 序列化。
- 代码生成。
- RPC 方法路由。
- status code。
- streaming 语义。

### “gRPC 一定比 REST 快”

也不准确。gRPC 经常有性能优势，但真实性能取决于：

- 请求大小。
- 连接是否复用。
- 服务端实现。
- 序列化成本。
- 网络环境。
- 是否启用压缩。

### “用了 gRPC 就不用写接口文档”

更不准确。proto 是机器可读的接口契约，但人仍然需要理解：

- 字段业务含义。
- 错误码约定。
- 鉴权方式。
- 超时策略。
- 版本兼容规则。

---

## 十二、本节验收

完成本节后，你应该能说清楚：

```text
RPC 是一种远程调用思想。
gRPC 是一种具体 RPC 框架。
Protobuf 是 gRPC 常用的契约和序列化方式。
HTTP/JSON 和 gRPC 不是谁完全替代谁，而是适合不同边界。
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