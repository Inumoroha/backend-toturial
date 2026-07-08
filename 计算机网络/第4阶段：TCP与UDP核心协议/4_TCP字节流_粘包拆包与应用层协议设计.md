# 4. TCP 字节流、粘包拆包与应用层协议设计

本节目标：理解 TCP 是字节流协议，不提供应用层消息边界，并能设计一个简单的长度前缀协议解决粘包拆包问题。

---

## 一、TCP 没有消息边界

你调用两次：

```go
conn.Write([]byte("hello"))
conn.Write([]byte("world"))
```

接收方不一定读到两次：

```text
hello
world
```

它可能一次读到：

```text
helloworld
```

也可能多次读到：

```text
hel
low
orld
```

这不是 TCP 错误，而是字节流特性。

---

## 二、什么是粘包和拆包

粘包：

```text
多次发送的数据被一次读到。
```

拆包：

```text
一次发送的数据被多次读到。
```

更准确地说，TCP 根本没有“包”的应用层概念。所谓粘包拆包，是应用层没有定义好消息边界。

---

## 三、常见解决方式

### 1. 固定长度

每条消息固定 128 字节。

简单，但浪费空间，不灵活。

### 2. 分隔符

每条消息用 `\n` 结尾。

适合文本协议，例如命令行聊天室。

### 3. 长度前缀

消息前 4 字节表示 body 长度。

适合 JSON、Protobuf、自定义二进制协议。

---

## 四、长度前缀写入

```go
func writeFrame(w io.Writer, payload []byte) error {
    var header [4]byte
    binary.BigEndian.PutUint32(header[:], uint32(len(payload)))

    if _, err := w.Write(header[:]); err != nil {
        return err
    }

    _, err := w.Write(payload)
    return err
}
```

解释：

- 前 4 字节写 body 长度。
- 使用大端序保持网络字节序习惯。
- 再写 body。

---

## 五、长度前缀读取

```go
func readFrame(r io.Reader) ([]byte, error) {
    var header [4]byte
    if _, err := io.ReadFull(r, header[:]); err != nil {
        return nil, err
    }

    n := binary.BigEndian.Uint32(header[:])
    if n > 1024*1024 {
        return nil, fmt.Errorf("frame too large: %d", n)
    }

    payload := make([]byte, n)
    if _, err := io.ReadFull(r, payload); err != nil {
        return nil, err
    }

    return payload, nil
}
```

关键点：

- `io.ReadFull` 保证读满指定长度。
- 必须限制最大消息大小。
- 不能假设一次 `Read` 就读完整消息。

---

## 六、Go 后端中的场景

你平时写 HTTP 不需要自己处理粘包，是因为 HTTP 协议已经定义了消息格式。

例如：

- Header 中有 `Content-Length`。
- Chunked 编码有块边界。
- HTTP/2 有 frame。

如果你自己基于 TCP 写协议，就必须自己定义边界。

---

## 补充实验：亲眼看到“两个 Write 不等于两个 Read”

服务端：

```go
package main

import (
    "log"
    "net"
)

func main() {
    ln, _ := net.Listen("tcp", ":9092")
    log.Println("listen on :9092")
    for {
        conn, _ := ln.Accept()
        go func(c net.Conn) {
            defer c.Close()
            buf := make([]byte, 16)
            for {
                n, err := c.Read(buf)
                if err != nil {
                    log.Println("read:", err)
                    return
                }
                log.Printf("read n=%d data=%q", n, buf[:n])
            }
        }(conn)
    }
}
```

客户端：

```go
package main

import (
    "log"
    "net"
)

func main() {
    conn, err := net.Dial("tcp", "127.0.0.1:9092")
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    conn.Write([]byte("hello"))
    conn.Write([]byte("world"))
}
```

你可能在服务端看到：

```text
read n=10 data="helloworld"
```

也可能看到：

```text
read n=5 data="hello"
read n=5 data="world"
```

两种结果都正常。TCP 只保证字节顺序，不保证你的 `Write` 边界会在对端 `Read` 中保留。

---

## 补充设计：一个协议字段至少要回答的问题

设计应用层协议时，不要只说“加长度前缀”。你至少要回答：

```text
长度字段几个字节？
使用大端还是小端？
长度是否包含 header 自己？
最大消息长度是多少？
消息体是 JSON、Protobuf 还是二进制？
读取到半包时如何等待？
长度非法时是否立刻断开？
协议版本如何升级？
```

一个简单但工程上可用的选择是：

```text
4 字节大端长度字段。
长度只表示 body。
body 使用 JSON。
最大长度 1MB。
超过最大长度直接关闭连接并记录日志。
```

第 9 小节的长度前缀协议完整实现，就是把这些选择落成代码。

---

## 七、常见问题

### 1. UDP 有粘包问题吗？

UDP 保留数据报边界，不是字节流。但 UDP 不保证可靠和顺序。

### 2. 为什么不能直接按缓冲区大小读？

缓冲区大小和应用消息大小没有必然关系。一次 Read 返回多少取决于内核缓冲、网络、调度等因素。

### 3. 长度前缀为什么要限制最大长度？

防止恶意或错误数据声明超大长度导致内存分配过大。

---

## 八、本节达标标准

学完本节后，你应该能够做到：

- 解释 TCP 字节流特性。
- 解释粘包拆包产生原因。
- 说出固定长度、分隔符、长度前缀三种边界方案。
- 写出长度前缀协议的读写函数。
- 解释为什么必须限制最大消息大小。
