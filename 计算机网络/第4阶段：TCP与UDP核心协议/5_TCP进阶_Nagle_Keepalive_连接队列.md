# 5. TCP 进阶：Nagle、Keepalive 与连接队列

本节目标：理解 Nagle 算法、TCP Keepalive、半连接队列、全连接队列，并知道它们如何影响 Go 后端服务。

---

## 一、Nagle 算法

Nagle 算法用于减少小包数量。

直觉是：

```text
如果前面的小数据还没确认，就先别急着发更多小数据。
```

优点：

- 减少小包。
- 提升网络利用率。

缺点：

- 对低延迟交互可能增加等待。

Go 中可以设置：

```go
tcpConn.SetNoDelay(true)
```

很多 Go TCP 连接默认已经禁用 Nagle，但你需要知道这个选项的含义。

---

## 二、TCP Keepalive

TCP Keepalive 用于探测空闲连接是否仍然存活。

适合场景：

- 长连接。
- 对端异常断电。
- NAT 或防火墙清理空闲连接。

Go 设置：

```go
tcpConn.SetKeepAlive(true)
tcpConn.SetKeepAlivePeriod(30 * time.Second)
```

注意区分：

```text
TCP Keepalive：传输层探活。
HTTP Keep-Alive：HTTP 连接复用。
应用层心跳：业务协议自己发 ping/pong。
```

---

## 三、半连接队列

服务端收到 SYN 并回复 SYN-ACK 后，连接还没完全建立，会进入半连接队列。

如果大量 SYN 进来，半连接队列可能满。

相关问题：

- SYN Flood。
- 新连接建立慢。
- 客户端连接超时。

查看参数：

```bash
sysctl net.ipv4.tcp_max_syn_backlog
```

---

## 四、全连接队列

三次握手完成后，连接进入全连接队列，等待应用 `Accept`。

如果应用 Accept 太慢，全连接队列可能积压。

查看监听：

```bash
ss -lnt
```

监听 socket 的 `Recv-Q`、`Send-Q` 可作为队列观察线索。

---

## 五、Go 服务中的影响

如果 Go 服务 CPU 打满、阻塞严重或 accept 处理异常，可能导致新连接建立困难。

常见应对：

- 保持 accept 循环简单。
- handler 中避免长期阻塞。
- 设置超时。
- 做限流和过载保护。
- 用压测观察连接状态。

---

## 六、实验：观察系统参数

```bash
sysctl net.core.somaxconn
sysctl net.ipv4.tcp_max_syn_backlog
ss -lnt
```

这些参数不要初学阶段随便修改。先学会看，再理解含义。

---

## 补充实验：观察监听队列和 Accept 变慢的影响

下面这个服务端故意让 `Accept` 后的处理变慢：

```go
package main

import (
    "log"
    "net"
    "time"
)

func main() {
    ln, err := net.Listen("tcp", ":9093")
    if err != nil {
        log.Fatal(err)
    }
    log.Println("listen on :9093")

    for {
        conn, err := ln.Accept()
        if err != nil {
            log.Println("accept:", err)
            continue
        }
        log.Println("accepted", conn.RemoteAddr())
        time.Sleep(2 * time.Second)
        go func() {
            defer conn.Close()
            time.Sleep(30 * time.Second)
        }()
    }
}
```

用多个终端快速连接：

```bash
for i in $(seq 1 50); do nc 127.0.0.1 9093 >/dev/null & done
```

观察：

```bash
ss -ant state established '( sport = :9093 )'
ss -lnt
```

你要理解：

```text
三次握手完成后的连接进入全连接队列，等待应用 accept。
应用 accept 太慢时，队列可能堆积。
队列满后，新连接可能变慢、失败或被丢弃。
```

Go 的 `net/http` 会持续 accept 连接，但如果业务处理、TLS 握手、连接数限制或系统参数不合理，仍然可能出现连接堆积。

---

## 补充理解：Keepalive 和应用心跳不是一回事

TCP Keepalive 是操作系统层面的探测，默认间隔通常很长，适合清理“对端已经消失但本端不知道”的连接。

应用心跳是业务协议的一部分，例如：

```json
{"type":"ping"}
```

它适合判断：

```text
这个用户是否还在线。
这个 WebSocket 连接是否可用。
这个业务会话是否应该清理。
```

后端工程里常见组合是：

```text
TCP Keepalive 作为底层兜底。
应用心跳作为业务层存活判断。
```

不要指望默认 TCP Keepalive 帮你做实时在线状态管理。

---

## 七、常见问题

### 1. TCP Keepalive 能替代业务心跳吗？

不完全能。业务心跳更可控，可以携带业务状态，检测周期也更灵活。

### 2. Nagle 一定要关闭吗？

不一定。低延迟小消息场景可能关闭，大批量传输场景不一定需要。

### 3. 全连接队列满一定是内核问题吗？

不一定，也可能是应用层 accept 或处理太慢。

---

## 八、本节达标标准

学完本节后，你应该能够做到：

- 解释 Nagle 算法的目的。
- 区分 TCP Keepalive、HTTP Keep-Alive 和应用心跳。
- 解释半连接队列和全连接队列。
- 使用 `sysctl` 和 `ss` 查看相关参数与监听状态。
