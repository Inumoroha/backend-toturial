# 01-Go TCP 编程教程

## 学习目标

掌握 Go 中 `net.Listener`、`net.Conn` 的基本用法，写出一个可并发处理连接的 TCP Echo Server。

## 一、Go TCP 编程模型

服务端基本流程：

1. `net.Listen` 监听地址。
2. 循环 `Accept` 接收连接。
3. 为每个连接启动 goroutine。
4. 从连接读取数据。
5. 向连接写入数据。
6. 连接结束后关闭。

客户端基本流程：

1. `net.DialTimeout` 建立连接。
2. 写入请求。
3. 读取响应。
4. 关闭连接。

## 二、TCP Echo Server

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

    log.Println("tcp echo server listen on :9000")

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

        fmt.Fprintln(conn, "echo:", msg)
    }
}
```

## 三、测试服务

使用 `nc`：

```bash
nc 127.0.0.1 9000
```

输入：

```text
hello
quit
```

## 四、TCP Client

```go
package main

import (
    "bufio"
    "fmt"
    "net"
    "time"
)

func main() {
    conn, err := net.DialTimeout("tcp", "127.0.0.1:9000", 3*time.Second)
    if err != nil {
        panic(err)
    }
    defer conn.Close()

    conn.SetDeadline(time.Now().Add(5 * time.Second))

    fmt.Fprintln(conn, "hello")

    resp, err := bufio.NewReader(conn).ReadString('\n')
    if err != nil {
        panic(err)
    }

    fmt.Print(resp)
}
```

## 五、关键点解释

### 为什么每个连接一个 goroutine

`Accept` 接收连接后，如果直接在主循环里处理读写，就无法继续接收新连接。用 goroutine 可以让多个连接并发处理。

Go runtime 底层使用 netpoller 管理网络 I/O，大量 goroutine 等待网络事件时不会简单等于大量系统线程。

### 为什么要设置 Deadline

如果客户端连接后不发送数据，服务端 `ReadString` 可能一直阻塞。设置 `SetReadDeadline` 可以避免连接长期占用资源。

### 为什么要关闭连接

连接是系统资源。忘记关闭会导致文件描述符泄漏，最终可能出现：

```text
too many open files
```

## 六、排查命令

查看监听：

```bash
ss -lntp | grep 9000
```

查看连接：

```bash
ss -antp | grep 9000
```

抓包：

```bash
sudo tcpdump -i any tcp port 9000 -nn
```

## 七、练习题

1. `net.Listener` 和 `net.Conn` 分别代表什么？
2. 为什么服务端要在循环中调用 `Accept`？
3. 为什么要为连接设置超时？
4. 如果不 `defer conn.Close()` 会有什么风险？

## 八、验收标准

你能写出 TCP Echo Server 和 Client，并能用 `nc`、`ss`、`tcpdump` 验证连接状态和数据传输。

