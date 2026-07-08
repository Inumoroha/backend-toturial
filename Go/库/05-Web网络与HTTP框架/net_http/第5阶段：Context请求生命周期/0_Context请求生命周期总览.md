# 0. Context 请求生命周期总览

本阶段目标：理解 `r.Context()` 如何贯穿一次 HTTP 请求，并学会用 context 处理取消、超时和下游调用控制。

后端工程中，context 不是装饰参数。它表达的是：

```text
这个请求是否还活着？
这个操作是否已经超时？
这个调用是否应该停止？
```

如果你以后写数据库查询、HTTP Client、Redis 调用、消息队列消费，都会反复看到 `context.Context`。它不是 Go Web 的可选知识，而是后端工程中控制请求生命周期的主线。

---

## 一、本阶段要解决什么问题

你需要理解这些问题：

- Handler 里的 `r.Context()` 从哪里来？
- 客户端断开连接后，服务端如何知道？
- 下游调用为什么必须设置超时？
- 为什么不要在 Handler 里随便使用 `context.Background()`？
- `context.Value` 能不能传业务参数？

这些问题决定了你的服务能不能及时释放资源。

---

## 二、本阶段学习顺序

1. `r.Context()` 的来源。
2. 客户端断开连接后的取消传播。
3. `context.WithTimeout` 控制下游调用。
4. Handler、Service、Repository 之间如何传递 context。
5. 不滥用 `context.Value`。
6. 让 Todo API 响应取消和超时。

---

## 三、本阶段核心直觉

请求 context 应该沿着调用链向下传：

```text
Handler
-> Service
-> Store
-> DB / HTTP Client / Redis
```

不要在中途断开：

```go
context.Background()
```

因为这样下游任务就失去了“客户端已经取消”或“请求已经超时”的信号。

---

## 四、本阶段达标标准

完成本阶段后，你应该能：

- 在 Handler 中使用 `r.Context()`。
- 用 `select` 监听 `ctx.Done()`。
- 用 `context.WithTimeout` 控制下游调用。
- 区分 `context.Canceled` 和 `context.DeadlineExceeded`。
- 解释 `context.Value` 的正确边界。
- 把 context 传入 Service 和 Store。
