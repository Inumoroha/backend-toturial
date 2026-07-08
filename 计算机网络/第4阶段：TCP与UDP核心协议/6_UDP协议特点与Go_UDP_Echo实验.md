# 6. UDP 协议特点与 Go UDP Echo 实验

本节目标：理解 UDP 的特点，并用 Go 写出 UDP Echo Server 和 Client。

---

## 一、UDP 的特点

UDP 是无连接协议。

特点：

- 不需要三次握手。
- 不保证可靠。
- 不保证顺序。
- 保留数据报边界。
- 头部开销小。
- 延迟低。

UDP 不是“不可靠所以没用”。DNS、QUIC、音视频、游戏同步都常见 UDP。

---

## 二、UDP 和 TCP 的区别

| 维度 | TCP | UDP |
| --- | --- | --- |
| 连接 | 面向连接 | 无连接 |
| 可靠性 | 保证可靠 | 不保证 |
| 顺序 | 保证有序 | 不保证 |
| 数据形式 | 字节流 | 数据报 |
| 典型场景 | HTTP、数据库、Redis | DNS、QUIC、音视频 |

---

## 三、UDP Server

```go
package main

import (
    "fmt"
    "net"
)

func main() {
    addr, err := net.ResolveUDPAddr("udp", ":9000")
    if err != nil {
        panic(err)
    }

    conn, err := net.ListenUDP("udp", addr)
    if err != nil {
        panic(err)
    }
    defer conn.Close()

    fmt.Println("udp listen on :9000")

    buf := make([]byte, 2048)
    for {
        n, clientAddr, err := conn.ReadFromUDP(buf)
        if err != nil {
            fmt.Println("read error:", err)
            continue
        }

        msg := string(buf[:n])
        fmt.Println("recv:", msg, "from", clientAddr)
        conn.WriteToUDP([]byte("echo: "+msg), clientAddr)
    }
}
```

---

## 四、UDP Client

```go
package main

import (
    "fmt"
    "net"
    "time"
)

func main() {
    addr, err := net.ResolveUDPAddr("udp", "127.0.0.1:9000")
    if err != nil {
        panic(err)
    }

    conn, err := net.DialUDP("udp", nil, addr)
    if err != nil {
        panic(err)
    }
    defer conn.Close()

    conn.SetDeadline(time.Now().Add(3 * time.Second))
    conn.Write([]byte("hello udp"))

    buf := make([]byte, 2048)
    n, err := conn.Read(buf)
    if err != nil {
        panic(err)
    }

    fmt.Println(string(buf[:n]))
}
```

---

## 五、抓包观察 UDP

```bash
sudo tcpdump -i any udp port 9000 -nn
```

运行 Client 后，你会看到 UDP 报文。不会看到 TCP 三次握手。

---

## 补充实验：给 UDP 加上请求编号

UDP 不保证可靠，也不保证应用层一定收到每个报文。为了观察这一点，给消息加编号。

客户端连续发送：

```go
package main

import (
    "fmt"
    "log"
    "net"
    "time"
)

func main() {
    conn, err := net.Dial("udp", "127.0.0.1:9000")
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    for i := 1; i <= 10; i++ {
        msg := fmt.Sprintf("seq=%02d", i)
        _, _ = conn.Write([]byte(msg))
        time.Sleep(100 * time.Millisecond)
    }
}
```

服务端收到后打印：

```text
seq=01
seq=02
seq=03
```

在本机环境通常不会丢包，但你仍然要在协议设计上假设：

```text
可能丢。
可能重复。
可能乱序。
可能对端根本没启动。
```

所以如果业务要求“必须送达”，你需要在 UDP 上层自己实现：

```text
请求编号。
ACK。
重传。
去重。
超时。
最大重试次数。
```

这就是为什么大多数普通后端 API 默认不用 UDP。

---

## 补充排障：UDP 服务是否真的收到包

UDP 没有连接，`curl` 这类 HTTP 工具不能直接测试 UDP 服务。

常用方法：

```bash
echo hello | nc -u 127.0.0.1 9000
```

抓包：

```bash
sudo tcpdump -i any -nn 'udp port 9000'
```

判断：

```text
抓包看不到请求：客户端没发出，或目标地址/端口错。
抓包看到请求，服务端没日志：程序没监听、监听地址错、读取逻辑错。
服务端有日志，客户端没响应：服务端没有回包或回包路径有问题。
```

UDP 排障更依赖抓包，因为没有 TCP 状态机帮你记录连接状态。

---

## 六、常见问题

### 1. UDP 会粘包吗？

UDP 保留数据报边界，不是 TCP 字节流那种粘包问题。

### 2. UDP 消息可以无限大吗？

不建议。过大的 UDP 数据报可能被 IP 分片，任何一个分片丢失都会导致整个数据报不可用。

### 3. UDP 不可靠怎么办？

如果业务需要可靠性，可以在应用层实现确认、重传、序列号，或者直接使用 TCP/QUIC。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 解释 UDP 的特点。
- 写出 UDP Echo Server 和 Client。
- 使用 tcpdump 抓 UDP 报文。
- 说明 UDP 和 TCP 的主要区别。
