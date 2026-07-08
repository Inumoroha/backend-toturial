# 5. 阶段练习：画出 net/http 调用链

本节目标：用自己的图总结 `net/http` 服务端和客户端主流程。

---

## 一、服务端调用链

请画出并解释：

```text
TCP connection
-> http.Server
-> ServeMux
-> Middleware chain
-> Handler
-> ResponseWriter
-> Client
```

你需要能说明：

- Server 负责监听和连接处理。
- ServeMux 负责路由匹配。
- Middleware 负责横切逻辑。
- Handler 负责业务 HTTP 输入输出。
- ResponseWriter 写回响应。

---

## 二、客户端调用链

请画出并解释：

```text
Your code
-> http.Client
-> Transport
-> TCP/TLS connection
-> Remote server
-> Response
-> Body.Close
```

你需要能说明：

- Client 应该复用。
- 请求应该带 context。
- Transport 管理连接池。
- 非 2xx 状态码要自己处理。
- Body 必须关闭。

---

## 三、源码阅读任务

打开 Go 标准库源码，找到：

- `type Handler interface`
- `type HandlerFunc`
- `type ServeMux`
- `type Server`
- `type Client`
- `type Transport`
- `type RoundTripper`

每看到一个类型，写一句它的职责。

---

## 四、阶段复盘

完成本阶段后，请确认你能回答：

- 为什么 Handler 是 `net/http` 的中心？
- ServeMux 为什么也是 Handler？
- Server 从连接到 Handler 大致经历什么？
- Client 和 Transport 如何分工？
- 源码阅读时为什么先抓主流程，而不是陷入所有细节？

下一阶段进入项目实战：Todo API 服务。

