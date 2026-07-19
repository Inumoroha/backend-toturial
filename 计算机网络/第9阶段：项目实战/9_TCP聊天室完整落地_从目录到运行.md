# 9. TCP 聊天室完整落地：从目录到运行

本节目标：把前面关于 TCP、粘包拆包、连接管理、心跳和广播的知识串起来，完成一个可以在本机运行的 TCP 聊天室。

这一节采用“先能跑，再解释，再扩展”的方式。你会写出服务端和客户端，并用多个终端验证广播效果。

---

## 一、项目目标

最终目录：

```text
tcp-chat/
  go.mod
  cmd/server/main.go
  cmd/client/main.go
  internal/protocol/frame.go
```

支持的命令：

```text
/name alice       设置昵称
/quit             退出聊天室
普通文本           广播给其他在线用户
```

协议格式：

```text
4 字节长度前缀 + JSON 消息体
```

消息体示例：

```json
{
  "type": "chat",
  "name": "alice",
  "body": "hello"
}
```

为什么不用 `\n` 分隔？

因为后端项目里经常需要传 JSON、二进制或多行文本。长度前缀协议更接近真实应用层协议设计。

---

## 二、创建项目

```powershell
mkdir tcp-chat
cd tcp-chat
go mod init example.com/tcp-chat
mkdir cmd
mkdir cmd\server
mkdir cmd\client
mkdir internal
mkdir internal\protocol
```

项目分层很简单：

```text
protocol     只负责消息编码和解码。
server       负责监听端口、管理连接、广播消息。
client       负责连接服务端、读终端、显示消息。
```

---

## 三、实现协议层

创建文件：

```text
internal/protocol/frame.go
```

写入：

```go
package protocol

import (
    "encoding/binary"
    "encoding/json"
    "errors"
    "fmt"
    "io"
)

const MaxFrameSize = 64 * 1024

type Message struct {
    Type string `json:"type"`
    Name string `json:"name,omitempty"`
    Body string `json:"body,omitempty"`
}

func WriteMessage(w io.Writer, msg Message) error {
    data, err := json.Marshal(msg)
    if err != nil {
        return err
    }
    if len(data) > MaxFrameSize {
        return fmt.Errorf("frame too large: %d", len(data))
    }

    var header [4]byte
    binary.BigEndian.PutUint32(header[:], uint32(len(data)))

    if _, err := w.Write(header[:]); err != nil {
        return err
    }
    _, err = w.Write(data)
    return err
}

func ReadMessage(r io.Reader) (Message, error) {
    var header [4]byte
    if _, err := io.ReadFull(r, header[:]); err != nil {
        return Message{}, err
    }

    size := binary.BigEndian.Uint32(header[:])
    if size == 0 {
        return Message{}, errors.New("empty frame")
    }
    if size > MaxFrameSize {
        return Message{}, fmt.Errorf("frame too large: %d", size)
    }

    data := make([]byte, size)
    if _, err := io.ReadFull(r, data); err != nil {
        return Message{}, err
    }

    var msg Message
    if err := json.Unmarshal(data, &msg); err != nil {
        return Message{}, err
    }
    return msg, nil
}
```

关键点：

- TCP 是字节流，不保留消息边界，所以必须自己设计协议。
- 先读 4 字节长度，再按长度读 JSON。
- `io.ReadFull` 保证读够指定字节，不够就返回错误。
- `MaxFrameSize` 防止恶意客户端声明一个超大长度耗尽内存。
- 使用大端序是网络协议的常见约定。

---

## 四、实现服务端结构

创建文件：

```text
cmd/server/main.go
```

先写结构定义：

```go
package main

import (
    "errors"
    "flag"
    "io"
    "log"
    "net"
    "strings"
    "sync"

    "example.com/tcp-chat/internal/protocol"
)

type client struct {
    conn net.Conn
    name string
    send chan protocol.Message
}

type server struct {
    mu      sync.Mutex
    clients map[*client]struct{}
}

func newServer() *server {
    return &server{
        clients: make(map[*client]struct{}),
    }
}
```

解释：

- 每个 TCP 连接对应一个 `client`。
- `send` 是该客户端的发送队列，避免广播时直接阻塞在慢客户端上。
- `server.clients` 保存所有在线客户端。
- `mu` 保护 `clients`，因为多个 goroutine 会同时读写它。

---

## 五、注册、注销和广播

继续写：

```go
func (s *server) add(c *client) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.clients[c] = struct{}{}
}

func (s *server) remove(c *client) {
    s.mu.Lock()
    defer s.mu.Unlock()
    if _, ok := s.clients[c]; ok {
        delete(s.clients, c)
        close(c.send)
        _ = c.conn.Close()
    }
}

func (s *server) broadcast(from *client, msg protocol.Message) {
    s.mu.Lock()
    defer s.mu.Unlock()

    for c := range s.clients {
        if c == from {
            continue
        }

        select {
        case c.send <- msg:
        default:
            log.Printf("drop slow client: %s", c.name)
        }
    }
}
```

这里最重要的是 `select default`。

如果你直接写：

```go
c.send <- msg
```

某个客户端一直不读消息时，整个广播会卡住。真实系统里慢客户端不能拖垮所有人。本项目采用简单策略：发送队列满了就丢消息并打印日志。

---

## 六、读循环和写循环

继续写：

```go
func (s *server) handleConn(conn net.Conn) {
    c := &client{
        conn: conn,
        name: conn.RemoteAddr().String(),
        send: make(chan protocol.Message, 16),
    }

    s.add(c)
    defer s.remove(c)

    go s.writeLoop(c)

    s.broadcast(c, protocol.Message{
        Type: "system",
        Body: c.name + " joined",
    })

    for {
        msg, err := protocol.ReadMessage(conn)
        if err != nil {
            if errors.Is(err, io.EOF) {
                log.Printf("client closed: %s", c.name)
            } else {
                log.Printf("read failed from %s: %v", c.name, err)
            }
            return
        }

        switch msg.Type {
        case "name":
            newName := strings.TrimSpace(msg.Name)
            if newName != "" {
                old := c.name
                c.name = newName
                s.broadcast(c, protocol.Message{
                    Type: "system",
                    Body: old + " is now " + c.name,
                })
            }
        case "chat":
            body := strings.TrimSpace(msg.Body)
            if body == "" {
                continue
            }
            s.broadcast(c, protocol.Message{
                Type: "chat",
                Name: c.name,
                Body: body,
            })
        default:
            log.Printf("unknown message type from %s: %s", c.name, msg.Type)
        }
    }
}

func (s *server) writeLoop(c *client) {
    for msg := range c.send {
        if err := protocol.WriteMessage(c.conn, msg); err != nil {
            log.Printf("write failed to %s: %v", c.name, err)
            return
        }
    }
}
```

读写拆成两个 goroutine 的原因：

- `readLoop` 只负责从客户端读消息。
- `writeLoop` 只负责向客户端写消息。
- 广播时只把消息放进 `send` 队列，不直接写 socket。

这是长连接服务常见结构。

---

## 七、服务端 main 函数

继续写：

```go
func main() {
    addr := flag.String("addr", ":9000", "listen address")
    flag.Parse()

    ln, err := net.Listen("tcp", *addr)
    if err != nil {
        log.Fatalf("listen failed: %v", err)
    }
    defer ln.Close()

    srv := newServer()
    log.Printf("tcp chat server listening on %s", *addr)

    for {
        conn, err := ln.Accept()
        if err != nil {
            log.Printf("accept failed: %v", err)
            continue
        }
        go srv.handleConn(conn)
    }
}
```

`Accept` 每收到一个连接，就开启一个 goroutine 处理。这个模型适合学习和中小规模连接数场景。

---

## 八、实现客户端

创建文件：

```text
cmd/client/main.go
```

写入：

```go
package main

import (
    "bufio"
    "flag"
    "fmt"
    "log"
    "net"
    "os"
    "strings"

    "example.com/tcp-chat/internal/protocol"
)

func main() {
    addr := flag.String("addr", "127.0.0.1:9000", "server address")
    flag.Parse()

    conn, err := net.Dial("tcp", *addr)
    if err != nil {
        log.Fatalf("dial failed: %v", err)
    }
    defer conn.Close()

    go readLoop(conn)

    scanner := bufio.NewScanner(os.Stdin)
    fmt.Println("connected. use /name alice, /quit or type message")

    for scanner.Scan() {
        line := strings.TrimSpace(scanner.Text())
        if line == "" {
            continue
        }
        if line == "/quit" {
            return
        }
        if strings.HasPrefix(line, "/name ") {
            name := strings.TrimSpace(strings.TrimPrefix(line, "/name "))
            _ = protocol.WriteMessage(conn, protocol.Message{
                Type: "name",
                Name: name,
            })
            continue
        }

        _ = protocol.WriteMessage(conn, protocol.Message{
            Type: "chat",
            Body: line,
        })
    }
}

func readLoop(conn net.Conn) {
    for {
        msg, err := protocol.ReadMessage(conn)
        if err != nil {
            log.Printf("server closed: %v", err)
            os.Exit(0)
        }

        switch msg.Type {
        case "system":
            fmt.Printf("[system] %s\n", msg.Body)
        case "chat":
            fmt.Printf("[%s] %s\n", msg.Name, msg.Body)
        default:
            fmt.Printf("[unknown] %+v\n", msg)
        }
    }
}
```

客户端也分成两个方向：

- 主 goroutine 从终端读取输入，写到 TCP 连接。
- `readLoop` 从 TCP 连接读取服务端广播，打印到终端。

如果只用一个 goroutine，客户端要么只能读终端，要么只能读网络，体验会很差。

---

## 九、运行验证

终端 1 启动服务端：

```powershell
go run ./cmd/server
```

终端 2 启动客户端 A：

```powershell
go run ./cmd/client
```

输入：

```text
/name alice
hello everyone
```

终端 3 启动客户端 B：

```powershell
go run ./cmd/client
```

输入：

```text
/name bob
hi alice
```

你应该看到：

- 客户端 A 能收到 bob 的消息。
- 客户端 B 能收到 alice 的消息。
- 服务端能看到连接加入、断开和错误日志。

---

## 十、抓包观察

启动服务端后，另开一个终端抓包：

```powershell
tcpdump -i any -nn tcp port 9000 -X
```

在 Windows 上如果没有 `tcpdump`，可以用 Wireshark 过滤：

```text
tcp.port == 9000
```

你会看到 TCP 载荷中先出现 4 字节长度，再出现 JSON 文本。注意：一次应用层消息不一定对应一个 TCP 包，所以不要通过“抓包里看起来一条一条的”来判断 TCP 保留消息边界。

---

## 十一、常见问题

### 1. 为什么客户端收不到自己发的消息？

服务端广播时跳过了发送者：

```go
if c == from {
    continue
}
```

这是聊天室的一种产品选择。如果希望自己也收到回显，可以删除这段判断。

### 2. 为什么要给 `send` 加缓冲？

缓冲队列可以吸收短时间的广播峰值。

如果没有缓冲，一个客户端慢一点就可能影响广播路径。缓冲不是万能的，所以队列满时仍然要有处理策略。

### 3. 这个聊天室能直接上生产吗？

不能。

它还缺认证、限流、心跳、空闲连接清理、房间隔离、消息持久化、指标和日志结构化。本节目标是把 TCP 长连接项目的骨架讲清楚。

### 4. 为什么协议层不直接用 `bufio.Scanner` 按行读？

按行协议简单，但限制也明显：

- 消息里不能自然包含换行。
- 默认 token 长度有限。
- 二进制内容不好处理。

长度前缀协议更通用。

---

## 十二、扩展练习

请在当前项目上继续完成：

1. 增加心跳消息 `ping/pong`。
2. 给每个连接设置 `ReadDeadline`，超过 60 秒没消息就断开。
3. 增加 `/who` 命令，返回在线用户列表。
4. 增加房间字段，只广播给同房间用户。
5. 给服务端增加优雅关闭。

---

## 十三、本节达标标准

学完本节后，你应该能够做到：

- 独立创建 TCP 聊天室项目目录。
- 实现 4 字节长度前缀协议。
- 解释为什么 TCP 需要应用层协议解决粘包拆包。
- 写出服务端 Accept 循环。
- 为每个连接维护读循环、写循环和发送队列。
- 使用互斥锁保护在线连接集合。
- 处理慢客户端对广播的影响。
- 写出可交互的命令行客户端。
- 用多个终端验证广播效果。
- 用抓包工具观察 TCP 载荷。

