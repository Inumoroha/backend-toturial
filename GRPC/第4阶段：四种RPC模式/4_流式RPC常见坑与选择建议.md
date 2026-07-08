# 4. 流式 RPC 常见坑与选择建议

本节目标：总结 Streaming RPC 的常见问题，并学会选择合适模式。

Streaming 很强大，但不是越多越好。很多业务用 Unary 更简单、更可靠。选择 RPC 模式时要从数据量、实时性、调用复杂度和错误处理出发。

---

## 一、核心直觉

- 数据量小、交互简单：优先 Unary。
- 服务端要逐步返回数据：Server Streaming。
- 客户端要连续上传数据：Client Streaming。
- 双方都需要实时交流：Bidirectional Streaming。
- 流式调用必须设计取消、超时、EOF、限速和错误策略。

---

## 二、动手步骤

1. 列出你的业务场景。
2. 判断数据方向：单条还是多条，谁多次发送。
3. 判断是否需要实时性。
4. 判断错误是立即失败，还是部分成功。
5. 根据结果选择 RPC 模式。

---

## 三、参考代码或命令

```text
导出 10 万条订单：Server Streaming
上传大文件分片：Client Streaming
在线客服聊天：Bidirectional Streaming
查询用户详情：Unary
创建订单：Unary
```

---

## 四、常见问题

- 为了炫技使用双向流：复杂度会明显增加。
- 忘记限速和背压：发送方太快会拖垮接收方。
- 错误语义不清：流中途失败，前面已经处理的数据如何回滚或记录？

---

## 五、练习任务

- 给 10 个业务场景选择 RPC 模式。
- 写出每个模式下的错误处理策略。
- 给一个 Server Streaming demo 增加 context 取消处理。

---

## 六、完成标准

- 能合理选择 RPC 模式。
- 知道流式 RPC 的主要风险。
- 不会滥用 streaming。


---

## 七、完整操作步骤：如何为业务选择 RPC 模式

选择 RPC 模式时，不要从技术炫技出发，而要从数据流向出发。

按下面步骤判断：

1. 客户端发送几次？一次还是多次？
2. 服务端返回几次？一次还是多次？
3. 是否需要实时性？
4. 是否允许中途部分成功？
5. 是否需要长期保持连接？
6. 出错后是否能重试？

根据答案选择：

```text
客户端一次 + 服务端一次 = Unary
客户端一次 + 服务端多次 = Server Streaming
客户端多次 + 服务端一次 = Client Streaming
客户端多次 + 服务端多次 = Bidirectional Streaming
```

---

## 八、业务场景对照表

| 场景 | 推荐模式 | 原因 |
| --- | --- | --- |
| 查询用户详情 | Unary | 一个 id，一个用户结果 |
| 创建订单 | Unary | 一次提交，一次返回 |
| 导出订单列表 | Server Streaming | 服务端逐条返回，避免大响应 |
| 查询任务进度 | Server Streaming | 服务端持续推送进度 |
| 上传大文件分片 | Client Streaming | 客户端连续发送分片 |
| 批量导入用户 | Client Streaming | 客户端连续提交，服务端汇总结果 |
| 在线聊天 | Bidirectional Streaming | 双方都随时发送消息 |
| 实时协作编辑 | Bidirectional Streaming | 客户端和服务端都推送事件 |
| 扣减库存 | Unary | 操作短小，必须有明确结果 |
| 推送订单状态 | Server Streaming | 客户端订阅，服务端推送 |

---

## 九、完整代码：模式选择伪代码

你可以在设计接口前写一个小决策函数，帮助自己判断：

```go
type RPCMode string

const (
    UnaryRPC       RPCMode = "Unary"
    ServerStream   RPCMode = "ServerStreaming"
    ClientStream   RPCMode = "ClientStreaming"
    BidiStream     RPCMode = "BidirectionalStreaming"
)

func chooseRPCMode(clientMany bool, serverMany bool) RPCMode {
    switch {
    case !clientMany && !serverMany:
        return UnaryRPC
    case !clientMany && serverMany:
        return ServerStream
    case clientMany && !serverMany:
        return ClientStream
    default:
        return BidiStream
    }
}
```

这不是生产代码，只是帮助你训练判断方式。

---

## 十、运行命令：用四个 demo 验证选择

如果你已经完成前面四种方法，可以分别运行：

```powershell
go run ./cmd/order-client unary
```

预期输出：

```text
order id=1 product=Book quantity=2 status=CREATED
```

```powershell
go run ./cmd/order-client server-stream
```

预期输出：

```text
order id=1 product=Book
order id=2 product=Keyboard
server stream finished
```

```powershell
go run ./cmd/order-client client-stream
```

预期输出：

```text
created count=3
```

```powershell
go run ./cmd/order-client bidi-stream
```

预期输出：

```text
recv: ack: client says order 1 paid
recv: ack: client says order 1 shipped
```

如果你没有实现命令行参数，也可以在 main 函数里依次调用四个函数。

---

## 十一、常见错误排查

### 1. 滥用双向流

双向流最复杂，不应该因为“高级”就优先使用。普通创建、查询、更新都应该先考虑 Unary。

### 2. 大列表仍然使用 Unary

如果响应可能非常大，例如导出 100 万条数据，不要一次性塞进 repeated response。可以使用 Server Streaming 分批返回。

### 3. 批量上传没有大小限制

Client Streaming 也需要限制：

- 最大消息数。
- 最大总字节数。
- 单条消息最大大小。
- 超时时间。

否则客户端可能把服务端拖垮。

### 4. 流式 RPC 没有取消处理

所有 streaming 服务端都应该关注：

```go
stream.Context().Done()
```

客户端断开后，服务端应该尽快停止工作。

### 5. 错误策略不清楚

Client Streaming 中，前面已经处理成功的消息，后面一条失败了怎么办？这必须在接口设计时说明。

---

## 十二、接口设计清单

设计 streaming RPC 前，先回答：

```text
[ ] 谁会发送多条消息？客户端、服务端，还是双方？
[ ] 流什么时候结束？
[ ] 中途出错如何处理？
[ ] 是否允许部分成功？
[ ] 是否需要幂等键？
[ ] 是否有最大消息数限制？
[ ] 是否有 deadline？
[ ] 客户端取消后服务端如何停止？
[ ] 是否需要鉴权 metadata？
[ ] 是否需要 trace id？
```

这个清单比直接写代码更重要。流式接口一旦设计混乱，后面很难维护。

---

## 十三、练习任务

为下面场景选择 RPC 模式，并写出原因：

1. 根据用户 id 查询用户信息。
2. 导出某个用户过去一年的订单。
3. 客户端上传 500MB 文件。
4. WebSocket 风格的客服聊天。
5. 创建支付单。
6. 服务端向客户端推送任务执行进度。
7. 客户端采集传感器数据，每秒上传一次，服务端每分钟汇总一次。
8. 双方实时同步协作文档编辑事件。

要求每个场景写成：

```text
场景：...
选择：...
原因：...
错误处理：...
是否需要超时/取消：...
```

---

## 十四、完成标准

完成本节后，你应该能：

```text
根据数据流向选择 RPC 模式
知道 Unary 是默认优先选择
知道什么时候使用 Server Streaming
知道什么时候使用 Client Streaming
知道什么时候才需要 Bidirectional Streaming
能说清每种模式的结束条件
能为 streaming 接口设计错误和取消策略
```

---

## 十五、阶段复盘

第 4 阶段完成后，请写一段复盘：

```markdown
# 四种 RPC 模式复盘

## Unary
适合：
不适合：
关键代码：

## Server Streaming
适合：
不适合：
关键代码：

## Client Streaming
适合：
不适合：
关键代码：

## Bidirectional Streaming
适合：
不适合：
关键代码：

## 我最容易混淆的地方
...
```

这个复盘会在后面学习错误处理、metadata、interceptor 时继续用到。

---

## 教程闭环检查

1. **完整操作步骤**：按业务数据流向选择 RPC 模式，并用四个 demo 验证。
2. **完整代码**：理解正文中的模式选择伪代码和前面四个 RPC demo。
3. **运行命令**：运行 unary、server-stream、client-stream、bidi-stream 四类调用。
4. **预期输出**：能看到每种模式不同的输出形态。
5. **常见错误排查**：重点检查滥用双向流、大列表 Unary、无取消处理、错误策略不清。
6. **练习任务**：完成八个业务场景的模式选择练习。
7. **完成标准**：能独立为真实业务选择合适的 RPC 模式，并说明理由。