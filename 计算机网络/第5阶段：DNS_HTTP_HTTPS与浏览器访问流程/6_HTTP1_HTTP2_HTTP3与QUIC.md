# 6. HTTP/1.1、HTTP/2、HTTP/3 与 QUIC

本节目标：理解 HTTP/1.1、HTTP/2、HTTP/3 的核心差异，以及 gRPC 为什么基于 HTTP/2。

---

## 一、HTTP/1.1

HTTP/1.1 支持长连接：

```text
多个请求可以复用同一个 TCP 连接。
```

但同一个连接上容易有应用层队头阻塞：

```text
前一个响应慢，后面的响应被挡住。
```

浏览器通常通过对同一域名开多个连接缓解。

---

## 二、HTTP/2

HTTP/2 引入：

- 二进制帧。
- 多路复用。
- Header 压缩。
- 流控制。

多个请求可以在同一个 TCP 连接上并发传输。

gRPC 使用 HTTP/2，因此天然支持：

- 多路复用。
- 流式调用。
- 强类型 RPC。

---

## 三、HTTP/2 仍有 TCP 层队头阻塞

HTTP/2 解决了 HTTP/1.1 应用层队头阻塞。

但它仍然基于 TCP。如果 TCP 丢包，同一个 TCP 连接上的多个 Stream 都可能受影响。

---

## 四、HTTP/3 与 QUIC

HTTP/3 基于 QUIC，QUIC 基于 UDP。

QUIC 提供：

- 可靠传输。
- TLS 1.3 集成。
- 更快连接建立。
- 多路复用。
- 连接迁移。

为什么用 UDP？

因为 TCP 在内核中演进慢，QUIC 在用户态基于 UDP 更容易迭代。

---

## 五、curl 观察 HTTP/2

```bash
curl -v --http2 https://example.com
```

关注：

- ALPN 是否协商到 `h2`。
- 响应协议版本。

---

## 补充实验：观察 ALPN 协商

HTTPS 连接中，客户端和服务端会通过 ALPN 协商使用哪个应用层协议。

执行：

```bash
curl -v --http2 https://example.com
```

关注输出中类似内容：

```text
ALPN: offers h2,http/1.1
ALPN: server accepted h2
using HTTP/2
```

这说明：

```text
客户端告诉服务端自己支持 h2 和 http/1.1。
服务端选择了 h2。
后续 HTTP 使用 HTTP/2。
```

如果服务端不支持 HTTP/2，可能会退回：

```text
using HTTP/1.1
```

所以“客户端支持 HTTP/2”不代表最终一定使用 HTTP/2，还要看服务端和中间代理是否支持。

---

## 补充理解：为什么 HTTP/2 适合 gRPC

gRPC 需要的不只是“发一个请求拿一个响应”，它还需要：

```text
强类型消息。
双向流。
多路复用。
Header 和 Trailer。
长连接复用。
```

HTTP/2 的 stream 模型刚好适合这些需求。一个 TCP 连接上可以同时有多个 stream：

```text
Stream 1：GetUser
Stream 3：ListOrders
Stream 5：WatchEvents
```

这比 HTTP/1.1 上排队请求更适合内部服务调用。

但你也要记住：

```text
HTTP/2 基于 TCP。
如果底层 TCP 丢包，多个 stream 可能一起受影响。
```

这就是 HTTP/3/QUIC 继续演进的重要原因之一。

---

## 补充对比：后端工程师真正关心什么

学习 HTTP 版本时，不要只背协议名，要能回答工程问题：

| 问题 | HTTP/1.1 | HTTP/2 | HTTP/3 |
| --- | --- | --- | --- |
| 多请求复用 | 长连接但并发能力弱 | 多路复用 | 多路复用 |
| 底层传输 | TCP | TCP | QUIC/UDP |
| TLS | 常见但不是协议强制 | 浏览器中通常配合 TLS | 默认集成 TLS 1.3 |
| 网关支持 | 最成熟 | 很成熟 | 仍受环境影响 |
| Go 后端常见性 | 很高 | gRPC 常见 | 逐步增加 |

实际选型时还要看：

```text
负载均衡是否支持。
网关是否支持。
客户端 SDK 是否支持。
抓包和排障工具是否成熟。
公司网络是否允许 UDP。
```

所以 HTTP/3 很先进，但不是所有内部系统都应该立刻迁移。

---

## 六、常见问题

### 1. HTTP/2 是否一定比 HTTP/1.1 快？

不一定。要看网络、服务端、代理、请求模式。

### 2. gRPC 能不能跑在 HTTP/1.1？

常见 gRPC-Go 依赖 HTTP/2。

### 3. HTTP/3 是不是 TCP？

不是。HTTP/3 基于 QUIC，QUIC 基于 UDP。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 解释 HTTP/1.1 长连接和队头阻塞。
- 解释 HTTP/2 多路复用。
- 说明 gRPC 与 HTTP/2 的关系。
- 解释 HTTP/3 和 QUIC 的基本关系。
