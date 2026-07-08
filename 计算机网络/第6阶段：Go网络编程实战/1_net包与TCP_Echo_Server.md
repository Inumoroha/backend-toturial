# 1. net 包与 TCP Echo Server

本节目标：使用 Go 标准库 `net` 写一个 TCP Echo Server，理解 `net.Listener`、`net.Conn`、`Accept`、`Read`、`Write`。

---

## 一、服务端流程

TCP Server 基本流程：

```text
Listen 监听端口。
Accept 接收连接。
每个连接启动 goroutine。
读取客户端数据。
写回响应。
关闭连接。
```

---

## 二、Echo Server

```go
package main

import (
    "bufio"
    "fmt"
    "log"
    "net"
    "strings"
    "time"
)

func main() {
    ln, err := net.Listen("tcp", ":9000")
    if err != nil {
        log.Fatal(err)
    }
    defer ln.Close()

    log.Println("tcp echo listen on :9000")

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

    reader := bufio.NewReader(conn)
    for {
        conn.SetReadDeadline(time.Now().Add(60 * time.Second))
        line, err := reader.ReadString('\n')
        if err != nil {
            log.Println("read error:", err)
            return
        }

        msg := strings.TrimSpace(line)
        if msg == "quit" {
            fmt.Fprintln(conn, "bye")
            return
        }

        conn.SetWriteDeadline(time.Now().Add(3 * time.Second))
        fmt.Fprintln(conn, "echo:", msg)
    }
}
```

---

## 三、测试

```bash
go run .
```

另一个终端：

```bash
nc 127.0.0.1 9000
```

输入：

```text
hello
quit
```

查看连接：

```bash
ss -antp | grep 9000
```

---

## 四、关键解释

`net.Listener` 表示监听 socket。

`Accept` 会阻塞等待新连接。

`net.Conn` 表示一条 TCP 连接，是双向字节流。

每个连接一个 goroutine 是 Go 中常见写法，但仍要设置超时和清理资源。

---

## 补充实验：同时连接多个客户端

启动服务端后，打开两个终端：

```bash
nc 127.0.0.1 9000
```

分别输入：

```text
hello from client 1
hello from client 2
```

观察服务端日志，你应该能看到两个不同的 `RemoteAddr`。

这一步验证：

```text
Accept 循环没有被单个连接阻塞。
每个连接都有自己的 goroutine。
一个客户端退出不会影响另一个客户端。
```

如果你把 `go handleConn(conn)` 改成 `handleConn(conn)`，再用两个客户端测试，会发现第二个连接可能要等第一个连接结束后才被处理。这就是为什么 TCP Server 通常要为每个连接启动独立处理逻辑。

---

## 补充排障：Echo Server 没响应怎么办

按顺序检查：

```bash
ss -lntp | grep 9000
nc -vz 127.0.0.1 9000
tcpdump -i any -nn tcp port 9000
```

判断：

```text
ss 看不到监听：服务没启动或端口错。
nc refused：端口没有监听。
tcpdump 有连接但服务端没日志：程序可能卡在 Accept 前或监听地址不是预期。
连接后不输入换行：ReadString('\n') 会一直等到换行。
```

这个小实验也能帮助你理解应用层协议边界：当前 Echo Server 以换行作为一条消息的结束标志。

---

## 五、常见问题

### 1. 为什么要 defer conn.Close？

连接是系统资源，不关闭会导致 fd 泄漏。

### 2. 为什么要 SetReadDeadline？

防止客户端连接后不发送数据，导致 goroutine 永久阻塞。

### 3. 为什么用换行分隔消息？

因为 TCP 没有消息边界，`\n` 是这里的应用层协议边界。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 写出 TCP Echo Server。
- 用 `nc` 连接测试。
- 解释 Listener、Conn、Accept 的作用。
- 说明为什么要设置 Deadline 和关闭连接。
