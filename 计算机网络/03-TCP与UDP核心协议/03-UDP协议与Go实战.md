# 03-UDP 协议与 Go 实战

## 学习目标

理解 UDP 的特点、适用场景，并能用 Go 写出 UDP Echo Server 和 Client。

## 一、UDP 的核心特点

UDP 是传输层协议，特点是：

- 无连接。
- 不保证可靠。
- 不保证顺序。
- 保留报文边界。
- 头部开销小。
- 延迟低。

UDP 不等于“不可靠所以没用”。它适合对实时性要求高，或者可靠性由应用层自己控制的场景。

## 二、UDP 与 TCP 对比

| 维度 | TCP | UDP |
| --- | --- | --- |
| 连接 | 面向连接 | 无连接 |
| 可靠性 | 保证可靠 | 不保证 |
| 顺序 | 保证有序 | 不保证 |
| 数据形式 | 字节流 | 数据报 |
| 典型场景 | HTTP/1.1、HTTP/2、数据库连接 | DNS、QUIC、音视频、游戏 |

## 三、报文边界

UDP 保留报文边界。发送方发一次，接收方通常按一次数据报读取。

但是 UDP 数据报过大可能被 IP 分片，丢一个分片就会导致整个数据报不可用。所以 UDP 消息应该控制大小。

## 四、Go UDP Server

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

    fmt.Println("udp server listen on :9000")

    buf := make([]byte, 2048)
    for {
        n, clientAddr, err := conn.ReadFromUDP(buf)
        if err != nil {
            fmt.Println("read error:", err)
            continue
        }

        msg := string(buf[:n])
        fmt.Println("recv from", clientAddr, msg)

        _, err = conn.WriteToUDP([]byte("echo: "+msg), clientAddr)
        if err != nil {
            fmt.Println("write error:", err)
        }
    }
}
```

## 五、Go UDP Client

```go
package main

import (
    "fmt"
    "net"
    "time"
)

func main() {
    serverAddr, err := net.ResolveUDPAddr("udp", "127.0.0.1:9000")
    if err != nil {
        panic(err)
    }

    conn, err := net.DialUDP("udp", nil, serverAddr)
    if err != nil {
        panic(err)
    }
    defer conn.Close()

    conn.SetDeadline(time.Now().Add(3 * time.Second))

    _, err = conn.Write([]byte("hello udp"))
    if err != nil {
        panic(err)
    }

    buf := make([]byte, 2048)
    n, err := conn.Read(buf)
    if err != nil {
        panic(err)
    }

    fmt.Println(string(buf[:n]))
}
```

## 六、实验：抓 UDP 报文

```bash
sudo tcpdump -i any udp port 9000 -nn
```

启动 Server，再运行 Client，观察 UDP 报文。

## 七、Go 后端关联

Go 后端中 UDP 常见于：

- DNS 查询。
- 日志或指标上报。
- 自定义高实时性协议。
- QUIC 和 HTTP/3 的底层理解。

虽然日常业务接口大多是 HTTP/gRPC，但理解 UDP 有助于理解 DNS 和 QUIC。

## 八、练习题

1. UDP 为什么比 TCP 更适合一些实时场景？
2. UDP 是否完全没有可靠性？如果业务需要可靠怎么办？
3. UDP 为什么要控制单个报文大小？
4. DNS 为什么常用 UDP？

## 九、验收标准

你能写出 UDP Echo Server 和 Client，并能解释 UDP 和 TCP 的核心差异。

