# 07-项目一：TCP 聊天室完整实现教程

## 学习目标

从 0 到 1 实现一个可运行的 TCP 聊天室。完成后你会真正理解长连接、读写循环、连接清理、广播、慢客户端处理和 Go 并发模型。

## 一、最终效果

启动服务端：

```bash
go run ./cmd/server
```

用两个终端连接：

```bash
nc 127.0.0.1 9000
```

客户端 A：

```text
/name alice
hello
```

客户端 B：

```text
/name bob
hi
```

双方可以互相收到消息。

## 二、创建项目

```bash
mkdir tcp-chatroom
cd tcp-chatroom
go mod init tcp-chatroom
mkdir -p cmd/server internal/chat
```

目录：

```text
tcp-chatroom/
  go.mod
  cmd/
    server/
      main.go
  internal/
    chat/
      server.go
      client.go
```

## 三、定义 Client

创建 `internal/chat/client.go`：

```go
package chat

import "net"

type Client struct {
    Name string
    Conn net.Conn
    Send chan string
}
```

字段说明：

- `Name`：用户昵称。
- `Conn`：TCP 连接。
- `Send`：服务端发给该用户的消息队列。

为什么要有 `Send`？

不要让多个 goroutine 同时写同一个 TCP 连接。每个客户端只允许自己的 `writeLoop` 写连接，其他地方通过 channel 投递消息。

## 四、定义 Server

创建 `internal/chat/server.go`：

```go
package chat

import (
    "bufio"
    "fmt"
    "log"
    "net"
    "strings"
    "sync"
    "time"
)

type Server struct {
    Addr string

    mu      sync.Mutex
    clients map[*Client]struct{}
    names   map[string]*Client
}

func NewServer(addr string) *Server {
    return &Server{
        Addr:    addr,
        clients: make(map[*Client]struct{}),
        names:   make(map[string]*Client),
    }
}
```

这里用两个 map：

- `clients`：遍历所有在线连接，用于广播。
- `names`：通过昵称查找客户端，用于昵称去重和私聊扩展。

## 五、启动监听

继续在 `server.go` 添加：

```go
func (s *Server) ListenAndServe() error {
    ln, err := net.Listen("tcp", s.Addr)
    if err != nil {
        return err
    }
    defer ln.Close()

    log.Println("chat server listen on", s.Addr)

    for {
        conn, err := ln.Accept()
        if err != nil {
            log.Println("accept error:", err)
            continue
        }

        client := &Client{
            Name: "guest",
            Conn: conn,
            Send: make(chan string, 16),
        }

        s.addClient(client)
        go s.readLoop(client)
        go s.writeLoop(client)

        client.Send <- "欢迎进入聊天室，请使用 /name 你的昵称 设置昵称"
    }
}
```

关键点：

- `Accept` 必须持续循环。
- 每个连接启动读循环和写循环。
- `Send` 使用有缓冲 channel，避免广播时被单个慢客户端立刻卡住。

## 六、加入和移除客户端

继续添加：

```go
func (s *Server) addClient(c *Client) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.clients[c] = struct{}{}
}

func (s *Server) removeClient(c *Client) {
    s.mu.Lock()
    defer s.mu.Unlock()

    if _, ok := s.clients[c]; !ok {
        return
    }

    delete(s.clients, c)
    if c.Name != "" && s.names[c.Name] == c {
        delete(s.names, c.Name)
    }

    close(c.Send)
    c.Conn.Close()
}
```

注意：

- 清理必须从 map 删除。
- 关闭 `Send` 让写循环退出。
- 关闭 TCP 连接释放系统资源。

## 七、读循环

继续添加：

```go
func (s *Server) readLoop(c *Client) {
    defer func() {
        s.broadcast("system", fmt.Sprintf("%s 离开了聊天室", c.Name), c)
        s.removeClient(c)
    }()

    scanner := bufio.NewScanner(c.Conn)
    for {
        c.Conn.SetReadDeadline(time.Now().Add(5 * time.Minute))

        if !scanner.Scan() {
            break
        }

        line := strings.TrimSpace(scanner.Text())
        if line == "" {
            continue
        }

        if strings.HasPrefix(line, "/") {
            s.handleCommand(c, line)
            continue
        }

        s.broadcast(c.Name, line, c)
    }

    if err := scanner.Err(); err != nil {
        log.Println("read error:", err)
    }
}
```

读循环负责：

- 读取客户端输入。
- 处理命令。
- 普通消息广播。
- 读失败后清理连接。

## 八、写循环

继续添加：

```go
func (s *Server) writeLoop(c *Client) {
    for msg := range c.Send {
        c.Conn.SetWriteDeadline(time.Now().Add(3 * time.Second))
        _, err := fmt.Fprintln(c.Conn, msg)
        if err != nil {
            log.Println("write error:", err)
            s.removeClient(c)
            return
        }
    }
}
```

为什么要设置写超时？

如果客户端不读数据，服务端写操作可能阻塞。写超时可以避免慢客户端长期占用资源。

## 九、广播

继续添加：

```go
func (s *Server) broadcast(from, msg string, except *Client) {
    s.mu.Lock()
    defer s.mu.Unlock()

    text := fmt.Sprintf("[%s] %s", from, msg)
    for c := range s.clients {
        if c == except {
            continue
        }

        select {
        case c.Send <- text:
        default:
            go s.removeClient(c)
        }
    }
}
```

这里的 `default` 很重要：

- 如果某个客户端的发送队列满了，说明它消费太慢。
- 不应该让它拖慢所有用户。
- 可以主动断开慢客户端。

## 十、命令处理

继续添加：

```go
func (s *Server) handleCommand(c *Client, line string) {
    fields := strings.Fields(line)
    if len(fields) == 0 {
        return
    }

    switch fields[0] {
    case "/name":
        if len(fields) != 2 {
            c.Send <- "用法：/name 昵称"
            return
        }
        s.rename(c, fields[1])
    case "/users":
        c.Send <- s.userList()
    case "/quit":
        c.Send <- "bye"
        s.removeClient(c)
    default:
        c.Send <- "未知命令。可用命令：/name /users /quit"
    }
}

func (s *Server) rename(c *Client, name string) {
    s.mu.Lock()
    defer s.mu.Unlock()

    if _, exists := s.names[name]; exists {
        c.Send <- "昵称已被使用"
        return
    }

    old := c.Name
    if old != "" && s.names[old] == c {
        delete(s.names, old)
    }

    c.Name = name
    s.names[name] = c
    c.Send <- "昵称已设置为 " + name
}

func (s *Server) userList() string {
    s.mu.Lock()
    defer s.mu.Unlock()

    names := make([]string, 0, len(s.clients))
    for c := range s.clients {
        names = append(names, c.Name)
    }
    return "在线用户：" + strings.Join(names, ", ")
}
```

## 十一、入口 main.go

创建 `cmd/server/main.go`：

```go
package main

import (
    "log"
    "tcp-chatroom/internal/chat"
)

func main() {
    srv := chat.NewServer(":9000")
    log.Fatal(srv.ListenAndServe())
}
```

## 十二、运行测试

启动服务：

```bash
go run ./cmd/server
```

终端 A：

```bash
nc 127.0.0.1 9000
```

输入：

```text
/name alice
hello
/users
```

终端 B：

```bash
nc 127.0.0.1 9000
```

输入：

```text
/name bob
hi alice
```

## 十三、观察连接

```bash
ss -antp | grep 9000
```

抓包：

```bash
sudo tcpdump -i any tcp port 9000 -nn
```

你应该能看到 TCP 连接建立、数据传输和关闭。

## 十四、当前版本的不足

这个版本能运行，但仍有改进空间：

- `bufio.Scanner` 默认 token 最大 64K，长消息需要调整 buffer。
- `/quit` 中 `removeClient` 后读循环 defer 还会再次调用，需要让清理幂等，目前 map 判断已经兜底。
- `broadcast` 中持锁时投递 channel，虽然有 default，但更严谨的做法是复制客户端列表后释放锁再投递。
- 没有优雅关闭。
- 没有应用层心跳。
- 没有结构化协议。

这些不足正好是下一轮迭代任务。

## 十五、练习任务

1. 增加 `/msg 用户 消息` 私聊命令。
2. 增加心跳命令 `/ping`，服务端回复 `pong`。
3. 把普通文本协议改成 JSON 行协议。
4. 增加服务端优雅关闭。
5. 编写一个 Go Client 替代 `nc`。

## 十六、验收标准

你能从空目录创建项目，运行服务端，用两个 `nc` 客户端互发消息，并能解释读循环、写循环、广播、连接清理和慢客户端处理的设计。
