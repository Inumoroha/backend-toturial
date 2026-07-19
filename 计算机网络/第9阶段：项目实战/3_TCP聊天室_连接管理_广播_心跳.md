# 3. TCP 聊天室：连接管理、广播、心跳

本节目标：实现 TCP 聊天室的关键流程：Accept、readLoop、writeLoop、广播、连接清理和心跳思路。

---

## 一、服务启动流程

```text
Listen :9000
Accept 连接
创建 Client
加入 clients
启动 readLoop
启动 writeLoop
```

---

## 二、连接处理骨架

```go
func (s *Server) handle(conn net.Conn) {
    c := &Client{
        Name: "guest",
        Conn: conn,
        Send: make(chan string, 16),
    }

    s.addClient(c)
    go s.readLoop(c)
    go s.writeLoop(c)
}
```

---

## 三、广播

```go
func (s *Server) broadcast(msg string, except *Client) {
    s.mu.Lock()
    defer s.mu.Unlock()

    for c := range s.clients {
        if c == except {
            continue
        }
        select {
        case c.Send <- msg:
        default:
            go s.removeClient(c)
        }
    }
}
```

`default` 用来避免慢客户端阻塞所有人。

---

## 四、连接清理

```go
func (s *Server) removeClient(c *Client) {
    s.mu.Lock()
    defer s.mu.Unlock()

    if _, ok := s.clients[c]; !ok {
        return
    }
    delete(s.clients, c)
    close(c.Send)
    c.Conn.Close()
}
```

清理要幂等，因为读写循环都可能触发关闭。

---

## 五、心跳设计

可以约定：

```text
/ping
```

服务端回复：

```text
pong
```

长连接中需要心跳发现断线和空闲连接。

---

## 补充实现顺序：先连接管理，再广播

推荐顺序：

```text
1. Accept 新连接。
2. 创建 Client。
3. 注册到 Server.clients。
4. 启动 readLoop。
5. 启动 writeLoop。
6. 实现 broadcast。
7. 断开时 unregister。
```

不要在 `readLoop` 中直接遍历所有连接并写 socket。更稳的方式是：

```text
readLoop 只负责读和解析。
broadcast 只负责投递到每个 client.send。
writeLoop 只负责从 send channel 写到连接。
```

这样慢客户端最多堵住自己的发送队列，不会把整个聊天室广播拖住。

---

## 补充心跳验收

心跳不是“发个 ping 就完了”。至少要验证：

```text
正常客户端会 pong，连接保持。
客户端断网或进程退出后，服务端能清理连接。
超过心跳超时时间后，在线列表不再显示该用户。
服务端日志能看到清理原因。
```

建议字段：

```go
type Client struct {
    lastSeen time.Time
}
```

每次收到消息或 pong 时更新 `lastSeen`，后台定时扫描超时连接。

---

## 补充排障：广播卡住时先看慢客户端

如果聊天室出现“所有人都收不到消息”，先检查广播逻辑是不是直接写 socket。

危险写法：

```text
遍历所有连接。
对每个连接直接 conn.Write。
某个客户端网络慢，整个循环卡住。
```

推荐写法：

```text
广播只投递到 send channel。
writeLoop 单独负责写连接。
send channel 满了就丢弃或断开该客户端。
```

这也是长连接服务最重要的工程直觉之一。

---

## 六、常见问题

### 1. 为什么不能多个 goroutine 同时写一个 conn？

容易造成消息交错和并发问题。建议每个连接只有一个写循环。

### 2. Send channel 为什么要有缓冲？

缓冲能吸收短暂写入波动，但不能无限大，否则慢客户端会消耗内存。

### 3. removeClient 为什么要幂等？

读循环、写循环、心跳超时都可能触发关闭，重复关闭不应导致 panic。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 实现连接注册和清理。
- 解释 readLoop 和 writeLoop 分工。
- 实现广播。
- 说明慢客户端为什么需要处理。
- 设计简单心跳。
