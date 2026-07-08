# 4. Client 与 Transport 源码直觉

本节目标：理解 HTTP Client 侧的主线：`Client` 负责请求流程，`Transport` 负责底层连接。

---

## 一、Client.Do 大致做什么

简化流程：

```text
构造 Request
-> Client.Do
-> 检查请求
-> 交给 Transport RoundTrip
-> 获取 Response
-> 返回给调用方
```

关键点：拿到 `Response` 后，调用方负责关闭 `resp.Body`。

---

## 二、RoundTripper 接口

标准库中有一个重要接口：

```go
type RoundTripper interface {
	RoundTrip(*Request) (*Response, error)
}
```

`http.Transport` 实现了它。

这意味着你可以在测试或特殊场景中替换 Transport。

---

## 三、Transport 做什么

`Transport` 负责：

- 建立 TCP 连接。
- TLS 握手。
- 代理。
- 连接池。
- HTTP/1.1 keep-alive。
- 部分 HTTP/2 支持。

所以连接复用主要不是 Client 自己做，而是底层 Transport 做。

---

## 四、为什么 Body.Close 影响连接复用

如果响应体没有被读完或关闭，Transport 可能无法复用这条连接。

所以客户端代码必须：

```go
defer resp.Body.Close()
```

必要时读取并丢弃剩余响应体，再关闭。普通 JSON API 通常完整 Decode 后关闭即可。

---

## 五、本节检查点

请确认你能回答：

- `RoundTripper` 接口的作用是什么？
- `Transport` 主要负责什么？
- 为什么复用 Client 有助于连接复用？
- 为什么关闭响应体很重要？

