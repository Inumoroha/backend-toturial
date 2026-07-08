# 3. UDP Echo Server 与 Client

本节目标：用 Go 写 UDP Server 和 Client，理解 UDP 数据报和 TCP 字节流的代码差异。

---

## 一、UDP Server

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
        n, remote, err := conn.ReadFromUDP(buf)
        if err != nil {
            fmt.Println("read error:", err)
            continue
        }
        msg := string(buf[:n])
        conn.WriteToUDP([]byte("echo: "+msg), remote)
    }
}
```

---

## 二、UDP Client

```go
package main

import (
    "fmt"
    "net"
    "time"
)

func main() {
    addr, _ := net.ResolveUDPAddr("udp", "127.0.0.1:9000")
    conn, err := net.DialUDP("udp", nil, addr)
    if err != nil {
        panic(err)
    }
    defer conn.Close()

    conn.SetDeadline(time.Now().Add(3 * time.Second))
    conn.Write([]byte("hello"))

    buf := make([]byte, 2048)
    n, err := conn.Read(buf)
    if err != nil {
        panic(err)
    }
    fmt.Println(string(buf[:n]))
}
```

---

## 三、抓包

```bash
sudo tcpdump -i any udp port 9000 -nn
```

你不会看到 TCP 三次握手。

---

## 补充实验：客户端超时不代表服务端一定失败

把客户端 deadline 设置得很短：

```go
conn.SetDeadline(time.Now().Add(10 * time.Millisecond))
```

如果服务端稍慢，客户端会报超时。但这不一定代表服务端没有收到 UDP 报文。

用抓包确认：

```bash
sudo tcpdump -i any -nn udp port 9000
```

判断：

```text
抓包看到客户端请求：包已经发出。
服务端日志有 recv：服务端收到。
客户端超时：可能是服务端回包慢、回包丢失或 deadline 太短。
```

UDP 没有连接状态，排障更依赖日志和抓包。

---

## 补充设计：UDP 应用层要自己补哪些能力

如果业务要可靠，就要自己设计：

```text
seq：消息编号。
ack：确认收到。
retry：超时重发。
dedup：重复消息去重。
deadline：最大等待时间。
max size：限制数据报大小，避免 IP 分片。
```

这就是为什么普通后端服务优先用 TCP/HTTP/gRPC，而不是直接用 UDP。

---

## 补充检查：UDP Echo 是否真正完成闭环

本节验收不要只看客户端打印了一行 echo。至少确认：

```text
服务端监听的是 UDP，不是 TCP。
客户端目标端口和服务端一致。
tcpdump 能看到 UDP 请求和响应。
客户端 deadline 生效。
服务端能打印 clientAddr。
```

如果服务端收到了请求但客户端没有收到响应，重点检查：

```text
WriteToUDP 是否使用了 ReadFromUDP 返回的 clientAddr。
客户端 deadline 是否太短。
防火墙是否拦截 UDP 回包。
```

---

## 补充练习：改成大写 Echo

把服务端返回值从原样 echo 改成大写：

```text
hello -> HELLO
```

这能确认你真正理解了读取、处理、回写三步。完成后再用 `tcpdump` 抓一次包，观察 UDP 请求和响应仍然是两个独立数据报。

---

## 四、常见问题

### 1. UDP Server 需要 Accept 吗？

不需要。UDP 无连接，直接从 socket 读取数据报。

### 2. UDP 是否需要设置超时？

需要。否则读取可能一直阻塞。

### 3. UDP 是否适合普通业务接口？

通常不适合。普通业务接口优先 HTTP/gRPC。

---

## 五、本节达标标准

学完本节后，你应该能够做到：

- 写出 UDP Server 和 Client。
- 使用 tcpdump 抓 UDP 报文。
- 解释 UDP 和 TCP 在 Go 编程模型上的差异。
