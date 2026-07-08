# 9. TCP 长度前缀协议完整实现

本节目标：完整实现一个基于 TCP 的长度前缀协议，解决粘包拆包问题。你会写出 codec、server、client，并用多个请求验证协议边界。

---

## 一、为什么要做这个实验

TCP 是字节流，不保留应用消息边界。

如果你自己基于 TCP 写协议，就必须回答：

```text
一条消息从哪里开始？
一条消息到哪里结束？
接收方如何知道 body 长度？
消息太大怎么办？
读到一半连接断了怎么办？
```

本节用最常见的长度前缀方案：

```text
4 字节长度 + JSON body
```

---

## 二、项目结构

创建目录：

```bash
mkdir tcp-frame-lab
cd tcp-frame-lab
go mod init tcp-frame-lab
mkdir codec server client
```

目录：

```text
tcp-frame-lab/
  go.mod
  codec/
    frame.go
  server/
    main.go
  client/
    main.go
```

---

## 三、实现 codec

创建 `codec/frame.go`：

```go
package codec

import (
    "encoding/binary"
    "fmt"
    "io"
)

const MaxFrameSize = 1024 * 1024

func WriteFrame(w io.Writer, payload []byte) error {
    if len(payload) > MaxFrameSize {
        return fmt.Errorf("frame too large: %d", len(payload))
    }

    var header [4]byte
    binary.BigEndian.PutUint32(header[:], uint32(len(payload)))

    if _, err := w.Write(header[:]); err != nil {
        return err
    }

    _, err := w.Write(payload)
    return err
}

func ReadFrame(r io.Reader) ([]byte, error) {
    var header [4]byte
    if _, err := io.ReadFull(r, header[:]); err != nil {
        return nil, err
    }

    n := binary.BigEndian.Uint32(header[:])
    if n > MaxFrameSize {
        return nil, fmt.Errorf("frame too large: %d", n)
    }

    payload := make([]byte, n)
    if _, err := io.ReadFull(r, payload); err != nil {
        return nil, err
    }

    return payload, nil
}
```

关键解释：

- 前 4 字节表示 body 长度。
- 使用 BigEndian，符合网络字节序习惯。
- `io.ReadFull` 保证读满指定长度。
- `MaxFrameSize` 防止恶意长度导致内存分配过大。

---

## 四、定义消息格式

本实验用 JSON：

```json
{
  "type": "echo",
  "body": "hello"
}
```

服务端收到后返回：

```json
{
  "type": "echo_reply",
  "body": "hello"
}
```

JSON 只是 body 格式，真正解决边界的是前面的 4 字节长度。

---

## 五、实现 Server

创建 `server/main.go`：

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net"
    "time"

    "tcp-frame-lab/codec"
)

type Message struct {
    Type string `json:"type"`
    Body string `json:"body"`
}

func main() {
    ln, err := net.Listen("tcp", ":9000")
    if err != nil {
        log.Fatal(err)
    }
    defer ln.Close()

    log.Println("frame server listen on :9000")

    for {
        conn, err := ln.Accept()
        if err != nil {
            log.Println("accept error:", err)
            continue
        }
        go handleConn(conn)
    }
}

func handleConn(conn net.Conn) {
    defer conn.Close()
    log.Println("new connection:", conn.RemoteAddr())

    for {
        conn.SetReadDeadline(time.Now().Add(60 * time.Second))
        payload, err := codec.ReadFrame(conn)
        if err != nil {
            log.Println("read frame error:", err)
            return
        }

        var req Message
        if err := json.Unmarshal(payload, &req); err != nil {
            log.Println("json error:", err)
            return
        }

        log.Printf("recv type=%s body=%s", req.Type, req.Body)

        resp := Message{
            Type: "echo_reply",
            Body: fmt.Sprintf("server received: %s", req.Body),
        }

        data, err := json.Marshal(resp)
        if err != nil {
            log.Println("marshal error:", err)
            return
        }

        conn.SetWriteDeadline(time.Now().Add(3 * time.Second))
        if err := codec.WriteFrame(conn, data); err != nil {
            log.Println("write frame error:", err)
            return
        }
    }
}
```

---

## 六、实现 Client

创建 `client/main.go`：

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
    "net"
    "time"

    "tcp-frame-lab/codec"
)

type Message struct {
    Type string `json:"type"`
    Body string `json:"body"`
}

func main() {
    conn, err := net.DialTimeout("tcp", "127.0.0.1:9000", 3*time.Second)
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    for i := 1; i <= 3; i++ {
        req := Message{
            Type: "echo",
            Body: fmt.Sprintf("hello-%d", i),
        }

        data, err := json.Marshal(req)
        if err != nil {
            log.Fatal(err)
        }

        conn.SetWriteDeadline(time.Now().Add(3 * time.Second))
        if err := codec.WriteFrame(conn, data); err != nil {
            log.Fatal(err)
        }

        conn.SetReadDeadline(time.Now().Add(3 * time.Second))
        payload, err := codec.ReadFrame(conn)
        if err != nil {
            log.Fatal(err)
        }

        var resp Message
        if err := json.Unmarshal(payload, &resp); err != nil {
            log.Fatal(err)
        }

        fmt.Printf("response: type=%s body=%s\n", resp.Type, resp.Body)
    }
}
```

---

## 七、运行验证

终端 A：

```bash
go run ./server
```

终端 B：

```bash
go run ./client
```

预期输出：

```text
response: type=echo_reply body=server received: hello-1
response: type=echo_reply body=server received: hello-2
response: type=echo_reply body=server received: hello-3
```

服务端能连续收到 3 条消息，而不是把它们混成一条。

---

## 八、抓包观察

抓端口：

```bash
sudo tcpdump -i any tcp port 9000 -nn -X
```

运行 client。

你可以看到 TCP 传输的是字节流。是否一次显示多条应用消息，取决于抓包展示和内核发送情况，但应用层通过长度前缀能正确拆出消息。

---

## 九、故意制造错误：超大 frame

把 client 中 body 改成很大的字符串，超过 `MaxFrameSize`，应该看到：

```text
frame too large
```

这说明最大长度限制生效。

为什么必须限制？

如果攻击者发送：

```text
长度 = 4GB
```

服务端直接按长度分配内存，就可能被打爆。

---

## 十、故意制造错误：连接中途断开

如果客户端写了 header 但 body 没写完整，服务端的 `io.ReadFull` 会返回错误。

这正是我们希望的行为：

```text
消息不完整，不应该交给业务处理。
```

真实生产协议还需要：

- 错误码。
- 心跳。
- 版本号。
- 压缩标志。
- 请求 ID。
- 鉴权信息。

---

## 十一、常见问题

### 1. 为什么不用一次 Read 读取一条消息？

因为 TCP 是字节流，一次 Read 返回多少字节不等于一条应用消息。

### 2. 为什么长度字段用 4 字节？

4 字节能表达最大约 4GB 的长度。但实际必须加业务上限。本实验限制为 1MB。

### 3. 为什么 JSON 外面还要长度前缀？

JSON 只描述内容格式，不告诉 TCP 接收方一条 JSON 消息在哪里结束。

### 4. Protobuf 是否也需要边界？

需要。Protobuf 是序列化格式，不是传输边界协议。gRPC 使用 HTTP/2 frame 解决边界。

---

## 十二、本节达标标准

学完本节后，你应该能够做到：

- 从空目录创建 TCP frame 项目。
- 实现 `WriteFrame` 和 `ReadFrame`。
- 解释 `io.ReadFull` 的作用。
- 解释为什么要限制最大 frame。
- 运行 server/client 连续发送多条消息。
- 说明长度前缀如何解决 TCP 粘包拆包。

