# 2. TCP Client、Deadline 与连接生命周期

本节目标：写一个 TCP Client，理解连接建立、读写超时、主动关闭和错误处理。

---

## 一、Client 基本流程

```text
Dial 建立连接。
设置 Deadline。
写请求。
读响应。
关闭连接。
```

---

## 二、TCP Client 示例

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

---

## 三、Deadline 类型

```go
conn.SetDeadline(t)
conn.SetReadDeadline(t)
conn.SetWriteDeadline(t)
```

区别：

- `SetDeadline`：读写都生效。
- `SetReadDeadline`：只限制读。
- `SetWriteDeadline`：只限制写。

---

## 四、连接生命周期

Client 主动关闭：

```go
conn.Close()
```

底层会触发 TCP 关闭流程。主动关闭方可能进入 TIME_WAIT。

如果程序不关闭连接，会占用：

- fd。
- 本地端口。
- 内核连接状态。
- goroutine。

---

## 补充实验：分别观察连接超时和读超时

连接一个不存在的地址：

```go
conn, err := net.DialTimeout("tcp", "10.255.255.1:9000", 2*time.Second)
```

这用于观察建连阶段超时。

连接成功后设置读超时：

```go
conn.SetReadDeadline(time.Now().Add(2 * time.Second))
buf := make([]byte, 1024)
n, err := conn.Read(buf)
```

如果服务端一直不返回，读操作会超时。

这两个超时发生在不同阶段：

```text
DialTimeout：TCP 连接建立阶段。
ReadDeadline：连接建立后等待数据阶段。
WriteDeadline：连接建立后写数据阶段。
```

排障时要能说清楚“卡在建连”还是“卡在读响应”。

---

## 补充实践：连接生命周期记录日志

建议客户端练习时记录：

```text
开始 Dial。
Dial 成功，本地地址和远端地址。
发送请求。
开始等待响应。
读取成功或超时。
关闭连接。
```

示例：

```go
log.Println("local:", conn.LocalAddr(), "remote:", conn.RemoteAddr())
```

这些日志在线上排查连接池、超时和上游异常时很有价值。

---

## 补充检查：客户端退出前必须确认资源释放

客户端代码写完后，至少检查：

```text
连接成功后是否 defer conn.Close。
读写是否设置 deadline。
错误日志是否包含 remote address。
超时错误是否和普通 EOF 区分。
退出时服务端是否看到连接关闭。
```

可以用：

```bash
ss -antp | grep 9000
```

确认客户端退出后连接没有长期残留。这个习惯能帮助你避免在后续 HTTP Client、gRPC Client、数据库连接池中犯同类错误。

---

## 五、常见问题

### 1. DialTimeout 控制整个请求吗？

不是。它主要控制连接建立阶段。读写仍要设置 Deadline。

### 2. 读超时是不是连接一定坏了？

不一定。可能只是对方暂时没发数据，但通常需要按业务决定是否关闭。

### 3. TCP Client 能不能复用？

裸 TCP 是否复用取决于你的协议设计。HTTP Client 的连接复用由 Transport 管理。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 写出 TCP Client。
- 使用 DialTimeout 建立连接。
- 设置读写 Deadline。
- 解释连接关闭和 TIME_WAIT 的关系。
