# 01-项目一：TCP 聊天室

## 项目目标

实现一个基于 TCP 的命令行聊天室。这个项目用于巩固 TCP 长连接、goroutine、连接管理、消息广播、心跳和超时。

## 一、功能要求

基础功能：

- 多客户端同时连接。
- 客户端输入昵称。
- 用户加入时广播通知。
- 用户发送消息时广播给其他用户。
- 用户输入 `/quit` 退出。
- 服务端检测断开连接并清理。

进阶功能：

- 心跳检测。
- 空闲超时踢出。
- 私聊命令 `/msg 用户 消息`。
- 在线用户列表 `/users`。
- 服务端优雅关闭。

## 二、协议设计

先使用简单文本协议，每条消息以换行结束：

```text
LOGIN alice
SAY hello everyone
USERS
QUIT
```

服务端响应：

```text
OK welcome alice
MSG bob hello
ERR nickname already used
```

后续可以升级为长度前缀 + JSON。

## 三、核心结构设计

```go
type Client struct {
    Name string
    Conn net.Conn
    Send chan string
}

type Server struct {
    mu      sync.Mutex
    clients map[string]*Client
}
```

为什么需要 `Send chan string`？

- 避免多个 goroutine 同时写同一个连接。
- 每个客户端有自己的写循环。
- 广播时只需要把消息投递到 channel。

## 四、服务端流程

```text
Listen
  -> Accept
    -> 创建 Client
    -> 启动 readLoop
    -> 启动 writeLoop
    -> 加入 clients
```

读循环负责：

- 读取客户端命令。
- 解析命令。
- 调用服务端方法。

写循环负责：

- 从 `Send` channel 读取消息。
- 写入 TCP 连接。
- 设置写超时。

## 五、关键实现点

### 连接清理

任何读写错误都应该触发清理：

```go
func (s *Server) removeClient(name string) {
    s.mu.Lock()
    defer s.mu.Unlock()

    c, ok := s.clients[name]
    if !ok {
        return
    }
    delete(s.clients, name)
    close(c.Send)
    c.Conn.Close()
}
```

### 广播

```go
func (s *Server) broadcast(from, msg string) {
    s.mu.Lock()
    defer s.mu.Unlock()

    for name, c := range s.clients {
        if name == from {
            continue
        }
        select {
        case c.Send <- fmt.Sprintf("MSG %s %s", from, msg):
        default:
            // 发送队列满，说明客户端太慢，可以断开
        }
    }
}
```

## 六、测试方式

启动服务端：

```bash
go run ./cmd/server
```

使用多个终端连接：

```bash
nc 127.0.0.1 9000
```

输入：

```text
LOGIN alice
SAY hello
USERS
QUIT
```

## 七、验收标准

必须完成：

- 两个以上客户端能互相收到消息。
- 用户退出后不会继续出现在在线列表。
- 客户端异常断开后服务端能清理连接。
- 服务端没有明显 goroutine 泄漏。

加分项：

- 支持私聊。
- 支持心跳。
- 支持优雅关闭。
- 使用长度前缀 + JSON 协议重构。

