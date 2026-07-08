# 1. TCP 连接建立：三次握手与抓包观察

本节目标：理解 TCP 三次握手做了什么，并用 tcpdump 抓到 SYN、SYN-ACK、ACK。

---

## 一、TCP 为什么需要连接

TCP 是面向连接的协议。通信前，双方要先确认：

```text
客户端能发送和接收。
服务端能发送和接收。
双方初始序列号是什么。
```

这就是三次握手要解决的问题。

---

## 二、三次握手流程

```text
客户端 -> 服务端：SYN，seq = x
服务端 -> 客户端：SYN + ACK，seq = y，ack = x + 1
客户端 -> 服务端：ACK，ack = y + 1
```

握手完成后，双方进入 ESTABLISHED 状态。

---

## 三、为什么不是两次握手

两次握手时，服务端无法确认客户端是否收到了自己的 SYN-ACK。

如果网络中有旧的 SYN 报文延迟到达，服务端可能误以为客户端要建立新连接，从而浪费资源。

三次握手可以让双方都确认：

```text
我能发。
我能收。
对方也能发和收。
```

---

## 四、准备一个 Go HTTP 服务

创建 `main.go`：

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "ok")
    })

    log.Println("listen on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

启动：

```bash
go run .
```

---

## 五、抓包观察三次握手

另一个终端执行：

```bash
sudo tcpdump -i any tcp port 8080 -nn
```

再请求：

```bash
curl http://127.0.0.1:8080
```

tcpdump 中你可能看到：

```text
Flags [S]
Flags [S.]
Flags [.]
```

含义：

- `[S]`：SYN。
- `[S.]`：SYN + ACK。
- `[.]`：ACK。

---

## 六、Go 中什么时候发生三次握手

当你写：

```go
conn, err := net.Dial("tcp", "127.0.0.1:8080")
```

或者 HTTP Client 第一次连接某个服务时：

```go
resp, err := http.Get("http://127.0.0.1:8080")
```

底层都会建立 TCP 连接。

如果 HTTP 连接被复用，后续请求不一定重新握手。

---

## 补充实验：把三次握手和本机连接状态对上

只看 `tcpdump` 还不够，你还要能把抓包结果和系统连接状态对应起来。

先启动服务：

```bash
go run .
```

另开终端持续观察 8080 端口：

```bash
watch -n 0.5 "ss -antp | grep ':8080' || true"
```

再开一个终端抓包：

```bash
sudo tcpdump -i any -nn -tttt 'tcp port 8080'
```

最后请求服务：

```bash
curl -v http://127.0.0.1:8080/
```

你要把三份信息串起来看：

```text
curl 输出：什么时候 Connected。
tcpdump 输出：SYN、SYN-ACK、ACK 什么时候出现。
ss 输出：连接什么时候进入 ESTABLISHED。
```

一次典型结果可以这样理解：

```text
127.0.0.1.53012 > 127.0.0.1.8080: Flags [S], seq 1000
127.0.0.1.8080 > 127.0.0.1.53012: Flags [S.], seq 2000, ack 1001
127.0.0.1.53012 > 127.0.0.1.8080: Flags [.], ack 2001
```

逐行解释：

- 第一行：客户端临时端口 `53012` 向服务端 `8080` 发起连接。
- 第二行：服务端确认客户端的 SYN，所以 `ack = 1001`，同时发送自己的 SYN。
- 第三行：客户端确认服务端的 SYN，所以 `ack = 2001`。

注意：真实初始序列号由系统生成，不会固定从 `1000` 或 `2000` 开始。这里的数字只是为了方便理解。

如果你想看真实绝对序列号，可以加 `-S`：

```bash
sudo tcpdump -i any -nn -S 'tcp port 8080'
```

---

## 补充排障：SYN 发出后没有后续包怎么办

如果抓包只看到：

```text
client > server: Flags [S]
client > server: Flags [S]
client > server: Flags [S]
```

说明客户端一直在重传 SYN。常见原因按优先级排：

```text
目标 IP 路由不通。
安全组或防火墙丢弃。
目标机器没收到包。
目标端口被中间设备拦截。
```

如果看到：

```text
server > client: Flags [R.]
```

通常表示连接被拒绝，常见原因是目标机器可达，但端口没有进程监听。

这就是线上排障时 `timeout` 和 `refused` 的核心区别：

```text
timeout：更像包被丢了，或者对方没有回应。
refused：更像对方明确告诉你，这个端口不接受连接。
```

---

## 七、常见问题

### 1. 三次握手属于 HTTP 吗？

不是。三次握手属于 TCP。HTTP 运行在 TCP 连接之上。

### 2. curl 每次都会三次握手吗？

不一定。单独运行 curl 通常会新建连接，但 HTTP 客户端连接池可能复用已有 TCP 连接。

### 3. SYN 发出后没有 SYN-ACK 是什么问题？

可能是目标端口不通、防火墙丢包、路由问题或服务未监听。

---

## 八、本节达标标准

学完本节后，你应该能够做到：

- 画出 TCP 三次握手流程。
- 解释 SYN、SYN-ACK、ACK。
- 使用 tcpdump 抓到三次握手。
- 说明 Go 的 `net.Dial` 和 HTTP Client 何时触发 TCP 连接建立。
