# 4. 调试工具 grpcurl 与 buf 初识

本节目标：认识 gRPC 调试和 proto 管理工具，为后续工程化打基础。

写 HTTP 服务时可以用浏览器、curl、Postman 调试；写 gRPC 服务时，也需要类似的工具。`grpcurl` 可以像 curl 一样调用 gRPC 服务，`buf` 可以帮你管理 proto 格式、lint 和破坏性变更检查。

---

## 一、核心直觉

- `grpcurl` 依赖 server reflection 或本地 proto 文件来知道服务结构。
- reflection 适合开发和内部调试，生产环境是否开启要看安全要求。
- `buf` 可以统一 proto 格式、检查命名规范、检查接口兼容性。
- 工具不是第一阶段的重点，但越早知道，后面越顺。

---

## 二、动手步骤

1. 先了解 `grpcurl` 的作用，不急着安装也可以。
2. 后面服务开启 reflection 后，用 `grpcurl list` 查看服务。
3. 用 `grpcurl describe` 查看方法和消息。
4. 用 `grpcurl -d` 发送 JSON 请求调用 RPC。
5. 项目变大后，再引入 `buf.yaml` 管理 proto。

---

## 三、参考代码或命令

```bash
grpcurl -plaintext localhost:50051 list

grpcurl -plaintext localhost:50051 describe user.v1.UserService

grpcurl -plaintext -d '{"id": 1}' \
  localhost:50051 user.v1.UserService/GetUser
```

---

## 四、常见问题

- 没开 reflection 却直接 `grpcurl list`：会失败，需要开启 reflection 或指定 proto 文件。
- 把 grpcurl 的 JSON 当成真实传输格式：gRPC 线上传输还是 Protobuf，JSON 只是工具输入形式。
- 过早研究 buf 全部功能：先会 lint 和 generate，后面再学 breaking change。

---

## 五、练习任务

- 记录 `grpcurl list`、`describe`、调用 RPC 的三个典型命令。
- 思考为什么 gRPC 需要 reflection。
- 后面写完第一个 server 后，用 grpcurl 调一次。

---

## 六、完成标准

- 知道 grpcurl 用来做什么。
- 知道 reflection 和 proto 文件都能帮助调试。
- 知道 buf 是 proto 工程化工具。


---

## 七、为什么 gRPC 需要专门调试工具

HTTP 服务可以直接用浏览器或 curl 调试，因为请求格式就是文本：URL、Header、JSON。gRPC 默认传输 Protobuf 二进制数据，浏览器和普通 curl 不能直接看懂。

所以你需要两类工具：

- 调用工具：`grpcurl`、`evans`。
- proto 管理工具：`buf`。

本节先建立直觉，不要求立刻掌握所有功能。

---

## 八、grpcurl 的三种常用方式

### 方式 1：依赖 reflection

服务端开启 reflection 后，可以直接列出服务：

```powershell
grpcurl -plaintext localhost:50051 list
```

查看某个服务：

```powershell
grpcurl -plaintext localhost:50051 describe user.v1.UserService
```

调用方法：

```powershell
grpcurl -plaintext -d '{"id":1}' localhost:50051 user.v1.UserService/GetUser
```

### 方式 2：指定 proto 文件

如果服务端没有开启 reflection，可以指定 proto：

```powershell
grpcurl -plaintext `
  -proto proto/user/v1/user.proto `
  -d '{"id":1}' `
  localhost:50051 user.v1.UserService/GetUser
```

### 方式 3：带 metadata 调用

```powershell
grpcurl -plaintext `
  -H "authorization: Bearer dev-token" `
  -H "x-trace-id: trace-001" `
  -d '{"id":1}' `
  localhost:50051 user.v1.UserService/GetUser
```

这在测试鉴权和 trace id 时非常有用。

---

## 九、grpcurl 常见输出

成功时可能输出：

```json
{
  "user": {
    "id": "1",
    "name": "Tom",
    "email": "tom@example.com"
  }
}
```

注意：JSON 中 int64 可能显示为字符串，这是 Protobuf JSON 映射的正常现象。

错误时可能输出：

```text
ERROR:
  Code: NotFound
  Message: user not found
```

这比只看 Go client 的错误字符串更直观。

---

## 十、buf 的作用

`buf` 主要解决 proto 工程化问题：

- 格式化 proto。
- lint 检查命名规范。
- 生成代码。
- 检查 breaking change。

学习初期可以先不用 buf，但你要知道真实团队经常会用它约束 proto 质量。

一个典型 `buf.yaml`：

```yaml
version: v2
lint:
  use:
    - STANDARD
breaking:
  use:
    - FILE
```

---

## 十一、什么时候开启 reflection

开发环境：建议开启，调试方便。

内部测试环境：可以开启，但要限制访问范围。

生产环境：谨慎开启，因为它可能暴露服务名、方法名和消息结构。

开启方式：

```go
reflection.Register(grpcServer)
```

---

## 十二、练习任务

等你完成第 3 阶段的第一个 server 后，回来做这几个练习：

1. 使用 `grpcurl list` 查看服务列表。
2. 使用 `grpcurl describe` 查看 UserService。
3. 使用 `grpcurl -d` 调用 GetUser。
4. 故意传不存在的 id，观察错误 code。
5. 带上 metadata 调用一次。

---

## 十三、完成标准

你不需要现在就完全掌握 grpcurl 和 buf，但要能回答：

- 为什么普通 curl 不能直接调 gRPC？
- reflection 有什么作用？
- grpcurl 如何发送请求体？
- buf 主要解决什么问题？
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