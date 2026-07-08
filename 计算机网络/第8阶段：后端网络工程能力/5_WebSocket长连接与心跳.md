# 5. WebSocket 长连接与心跳

本节目标：理解 WebSocket 的用途、握手、长连接管理和心跳机制。

---

## 一、WebSocket 解决什么问题

HTTP 请求响应模型适合客户端主动请求。

WebSocket 适合：

- 即时聊天。
- 实时通知。
- 在线协作。
- 实时行情。

它允许客户端和服务端在同一连接上双向发送消息。

---

## 二、握手

WebSocket 从 HTTP Upgrade 开始：

```http
GET /ws HTTP/1.1
Upgrade: websocket
Connection: Upgrade
```

握手成功后，连接升级为 WebSocket 协议。

---

## 三、后端关注点

长连接服务要处理：

- 连接注册。
- 连接清理。
- 心跳。
- 写超时。
- 慢客户端。
- 广播。
- 在线数量。

不要只考虑 QPS，还要考虑同时在线连接数。

---

## 四、心跳

心跳用于发现连接是否还活着。

常见：

```text
ping -> pong
```

应用层心跳比单纯依赖 TCP Keepalive 更可控。

---

## 补充结构：长连接服务通常怎么组织

一个 WebSocket 服务通常至少有这些结构：

```text
Hub：管理所有连接。
Client：表示一个在线连接。
readLoop：读取客户端消息。
writeLoop：向客户端写消息。
send channel：每个客户端自己的发送队列。
```

伪代码：

```go
type Client struct {
    id   string
    send chan []byte
}

type Hub struct {
    register   chan *Client
    unregister chan *Client
    broadcast  chan []byte
    clients    map[*Client]struct{}
}
```

为什么要有 `send channel`？

```text
广播时不能直接阻塞在某个慢客户端的网络写入上。
每个客户端有自己的发送队列。
队列满时可以丢弃、断开或降级。
```

这和 TCP 聊天室项目里的连接管理思想是一致的。

---

## 补充实践：心跳参数怎么定

常见配置：

```text
服务端每 30 秒发送 ping。
客户端 10 秒内必须 pong。
连续 2-3 次失败断开连接。
写消息设置 write deadline。
读消息设置 read deadline。
```

设计时要考虑：

```text
心跳太频繁：在线连接多时心跳流量和 CPU 压力明显。
心跳太慢：断线发现不及时，在线状态不准确。
移动网络：偶发抖动更明显，不能太激进。
代理/NAT：空闲连接可能被中间设备清理。
```

所以心跳不是固定答案，要结合业务对“在线状态实时性”的要求来定。

---

## 补充排障：WebSocket 线上常见问题

常见现象：

```text
连接一段时间后自动断开。
服务端在线人数只增不减。
广播时某些用户收不到消息。
CPU 不高但内存持续上涨。
```

排查方向：

```text
代理是否支持 Upgrade。
Nginx proxy_read_timeout 是否太短。
连接关闭时是否从 Hub 移除。
慢客户端发送队列是否无限增长。
心跳失败是否及时断开。
fd 数量是否接近上限。
```

Nginx 代理 WebSocket 通常需要：

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

如果代理层没有正确处理 Upgrade，WebSocket 握手就可能失败。

---

## 五、常见问题

### 1. WebSocket 是否适合所有实时场景？

不一定。简单单向通知可以考虑 SSE；内部服务间流式通信可以考虑 gRPC stream。

### 2. TCP Keepalive 能不能替代 WebSocket 心跳？

通常不建议完全替代。应用层心跳周期和语义更可控。

### 3. 长连接服务主要瓶颈是 QPS 吗？

不只是 QPS，还要关注在线连接数、内存、fd、心跳频率和慢客户端。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 解释 WebSocket 和 HTTP Upgrade 的关系。
- 说明 WebSocket 适合哪些场景。
- 说出长连接服务必须处理的资源问题。
- 解释为什么需要心跳。
