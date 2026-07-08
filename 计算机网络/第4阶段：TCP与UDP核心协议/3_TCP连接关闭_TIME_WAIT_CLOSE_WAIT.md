# 3. TCP 连接关闭：TIME_WAIT 与 CLOSE_WAIT

本节目标：理解 TCP 四次挥手、TIME_WAIT、CLOSE_WAIT，并能用 `ss` 观察连接状态。

---

## 一、为什么关闭连接通常是四次

TCP 是双向连接。关闭时，两个方向要分别关闭。

典型流程：

```text
主动关闭方 -> 被动关闭方：FIN
被动关闭方 -> 主动关闭方：ACK
被动关闭方 -> 主动关闭方：FIN
主动关闭方 -> 被动关闭方：ACK
```

所以通常叫四次挥手。

---

## 二、TIME_WAIT

主动关闭连接的一方通常进入 TIME_WAIT。

作用：

- 确保最后一个 ACK 有机会被对方收到。
- 避免旧连接的延迟报文影响新连接。

查看：

```bash
ss -ant state time-wait
```

大量 TIME_WAIT 常见原因：

- 短连接太多。
- HTTP Client 没有复用连接。
- 代理层频繁主动关闭连接。

不要一看到 TIME_WAIT 就急着“优化掉”。先分析为什么短连接这么多。

---

## 三、CLOSE_WAIT

CLOSE_WAIT 通常表示：

```text
对方已经关闭连接，本地程序还没有关闭。
```

查看：

```bash
ss -ant state close-wait
```

大量 CLOSE_WAIT 常见于代码问题：

- HTTP 响应体没有关闭。
- TCP 读到 EOF 后没有关闭连接。
- goroutine 卡住，连接清理逻辑没有执行。

Go HTTP Client 常见错误：

```go
resp, err := client.Get(url)
if err != nil {
    return err
}
// 忘记 resp.Body.Close()
```

正确写法：

```go
resp, err := client.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close()
```

---

## 四、实验：观察连接状态

启动一个 HTTP 服务：

```bash
python3 -m http.server 8080
```

循环请求：

```bash
for i in {1..50}; do curl -s http://127.0.0.1:8080 > /dev/null; done
```

查看：

```bash
ss -ant | grep 8080
```

你可能看到 TIME_WAIT。

---

## 五、后端排障中的判断

| 状态 | 常见含义 |
| --- | --- |
| ESTABLISHED | 连接已建立 |
| TIME_WAIT | 主动关闭后等待 |
| CLOSE_WAIT | 对方关闭，本地未关闭 |
| SYN_SENT | 客户端发起连接，等待响应 |
| SYN_RECV | 服务端收到 SYN，等待 ACK |

如果线上 CLOSE_WAIT 持续增长，优先查代码连接关闭。

---

## 补充实验：主动关闭和被动关闭分别是谁

准备服务端：

```go
package main

import (
    "bufio"
    "log"
    "net"
)

func main() {
    ln, err := net.Listen("tcp", ":9090")
    if err != nil {
        log.Fatal(err)
    }
    log.Println("listen on :9090")

    for {
        conn, err := ln.Accept()
        if err != nil {
            log.Println(err)
            continue
        }
        go func(c net.Conn) {
            defer c.Close()
            line, err := bufio.NewReader(c).ReadString('\n')
            log.Printf("read line=%q err=%v", line, err)
        }(conn)
    }
}
```

启动后，用 `nc` 连接：

```bash
nc 127.0.0.1 9090
```

输入一行后退出 `nc`。观察：

```bash
ss -antp | grep 9090
```

谁先发送 FIN，谁就是主动关闭方。主动关闭方更容易进入 `TIME_WAIT`。

你也可以抓包：

```bash
sudo tcpdump -i any -nn 'tcp port 9090 and (tcp[tcpflags] & (tcp-fin|tcp-ack) != 0)'
```

重点看：

```text
哪一端先发 Flags [F.]。
另一端随后 ACK。
另一端再发 FIN。
先发 FIN 的一端最后 ACK，并进入 TIME_WAIT。
```

---

## 补充实验：故意制造 CLOSE_WAIT

下面这个服务端故意不关闭连接：

```go
package main

import (
    "io"
    "log"
    "net"
    "time"
)

func main() {
    ln, _ := net.Listen("tcp", ":9091")
    log.Println("listen on :9091")

    for {
        conn, _ := ln.Accept()
        go func() {
            _, _ = io.ReadAll(conn)
            log.Println("client closed, but server forgets Close")
            time.Sleep(10 * time.Minute)
        }()
    }
}
```

客户端连接后立刻退出：

```bash
echo hello | nc 127.0.0.1 9091
```

查看：

```bash
ss -antp | grep CLOSE-WAIT
```

如果看到服务端连接停在 `CLOSE-WAIT`，说明：

```text
对端已经关闭。
本端应用层还没有调用 Close。
```

修复方式就是确保所有连接、响应体、文件都在合适位置关闭：

```go
defer conn.Close()
defer resp.Body.Close()
```

---

## 六、常见问题

### 1. TIME_WAIT 一定是问题吗？

不是。它是 TCP 正常状态。大量、持续、影响端口资源时才需要分析。

### 2. CLOSE_WAIT 一定是本地问题吗？

通常说明本地应用没有及时关闭连接，优先查本地代码。

### 3. HTTP 长连接会不会减少 TIME_WAIT？

连接复用可以减少频繁建立和关闭连接，从而减少短连接带来的 TIME_WAIT 压力。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 画出 TCP 四次挥手。
- 解释 TIME_WAIT 的作用。
- 解释 CLOSE_WAIT 常见原因。
- 使用 `ss` 查看连接状态。
- 在 Go 代码中主动关闭响应体和 TCP 连接。
