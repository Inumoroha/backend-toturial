# 04-TCP 进阶：Nagle、Keepalive、连接队列与拥塞控制

## 学习目标

补齐 TCP 中更贴近后端工程的细节：Nagle 算法、TCP Keepalive、半连接队列、全连接队列、滑动窗口、流量控制、拥塞控制，以及它们如何影响 Go 服务。

## 一、滑动窗口

TCP 不会发送一个字节就等一个 ACK，否则吞吐会很低。滑动窗口允许发送方在未收到 ACK 前连续发送一定数量的数据。

你可以把窗口理解为：

```text
发送方当前最多可以“在路上”的数据量。
```

窗口越大，理论吞吐越高；窗口越小，发送方越容易停下来等待。

## 二、流量控制

流量控制保护接收方。

如果接收方应用程序读取很慢，接收缓冲区会被占满。接收方会通过 TCP 窗口告诉发送方：

```text
我现在接不下那么多了，你慢一点。
```

这就是接收窗口。

后端关联：

- 客户端读响应很慢，服务端写响应可能阻塞。
- Go 服务向慢客户端写大量数据时，要设置写超时。
- WebSocket 慢消费者必须处理，否则会拖垮服务端。

## 三、拥塞控制

拥塞控制保护网络。

即使接收方还能接收，网络中间链路也可能拥塞。TCP 会根据丢包、延迟、ACK 等信号调整发送速率。

常见阶段：

- 慢启动：刚开始逐步增大发送窗口。
- 拥塞避免：增长变慢，避免冲太猛。
- 快速重传：收到多个重复 ACK 后尽快重传。
- 快速恢复：拥塞后逐步恢复发送能力。

后端关联：

- 跨地域调用 RTT 高，慢启动影响更明显。
- 新建连接传大响应，不如复用已有连接稳定。
- 丢包会让 TCP 降速，表现为接口延迟升高或吞吐下降。

## 四、Nagle 算法

Nagle 算法用于减少小包数量。它的大致思想是：

```text
如果前面还有未确认的小数据，就先攒一攒，不要立刻发更多小包。
```

优点：

- 减少网络小包。
- 提升网络利用率。

缺点：

- 对低延迟交互可能造成额外等待。

典型影响场景：

- 聊天。
- 游戏。
- RPC 小请求。
- 需要快速响应的交互协议。

Go 中可以通过 TCPConn 设置：

```go
tcpConn, ok := conn.(*net.TCPConn)
if ok {
    tcpConn.SetNoDelay(true)
}
```

`SetNoDelay(true)` 表示禁用 Nagle，Go TCP 默认通常已经启用 NoDelay，但你应该理解这个选项的意义。

## 五、TCP Keepalive

TCP Keepalive 用于探测空闲连接是否仍然存活。

它解决的问题：

- 对端异常断电。
- 中间 NAT 或防火墙静默丢弃连接。
- 长连接空闲太久，但应用层不知道是否还可用。

注意区分：

- TCP Keepalive：传输层探活。
- HTTP Keep-Alive：HTTP 连接复用。
- 应用层心跳：业务协议自己发送 ping/pong。

Go 设置示例：

```go
tcpConn, ok := conn.(*net.TCPConn)
if ok {
    tcpConn.SetKeepAlive(true)
    tcpConn.SetKeepAlivePeriod(30 * time.Second)
}
```

对 WebSocket、TCP 聊天室、长连接网关来说，应用层心跳通常比只依赖 TCP Keepalive 更可控。

## 六、半连接队列与全连接队列

服务端监听 TCP 端口后，连接建立过程中涉及两个队列。

### 半连接队列

客户端发 SYN，服务端回 SYN-ACK，此时连接还没完全建立，处于半连接状态。

如果大量 SYN 进来，半连接队列可能满。

相关问题：

- SYN Flood。
- 服务端连接建立变慢。
- 客户端连接超时。

### 全连接队列

三次握手完成后，连接进入全连接队列，等待应用程序 `Accept`。

如果应用程序 Accept 太慢，全连接队列可能满。

后端关联：

- Go 服务 accept 循环被阻塞，会影响新连接接入。
- CPU 打满时，应用处理 accept 变慢。
- 突发流量下新连接可能失败。

Linux 参数：

```bash
sysctl net.core.somaxconn
sysctl net.ipv4.tcp_max_syn_backlog
```

查看监听队列：

```bash
ss -lnt
```

输出中的 `Recv-Q` 和 `Send-Q` 对监听 socket 有特殊含义，可用于观察队列积压线索。

## 七、TIME_WAIT 能不能直接优化掉

不要一看到 TIME_WAIT 就想“清理掉”。

TIME_WAIT 是 TCP 正常机制，用于：

- 确保最后 ACK 被对方收到。
- 防止旧连接报文污染新连接。

真正要分析的是：

- 为什么短连接这么多？
- 是否应该使用连接池？
- 是否复用 HTTP Client？
- 是否由客户端主动关闭导致 TIME_WAIT 集中在本机？

## 八、实验：观察连接队列和状态

启动一个慢处理服务：

```go
package main

import (
    "fmt"
    "net/http"
    "time"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        time.Sleep(3 * time.Second)
        fmt.Fprintln(w, "ok")
    })
    http.ListenAndServe(":8080", nil)
}
```

压测：

```bash
hey -n 1000 -c 200 http://127.0.0.1:8080/
```

观察连接：

```bash
ss -antp | grep 8080
ss -lnt | grep 8080
```

思考：

- 建立连接是否变慢？
- ESTABLISHED 是否堆积？
- 客户端是否出现超时？

## 九、实验：慢客户端写阻塞

服务端向客户端持续写大响应，客户端慢速读取时，服务端可能阻塞在写操作。生产中这会消耗 goroutine 和连接资源。

解决方式：

- 设置 `WriteTimeout`。
- 限制响应大小。
- 对流式连接做应用层心跳和发送队列上限。
- 慢消费者主动断开。

## 十、Go 后端关联

你应该把这些 TCP 细节映射到 Go 配置：

| TCP 问题 | Go 应对 |
| --- | --- |
| 连接长期空闲 | IdleTimeout、应用层心跳 |
| 客户端慢读 | WriteTimeout |
| 客户端慢发 Header | ReadHeaderTimeout |
| 短连接太多 | 复用 http.Client、调连接池 |
| 粘包拆包 | 设计应用层协议边界 |
| 连接泄漏 | Close、context、读写错误处理 |
| 下游过载 | 限流、熔断、连接池上限 |

## 十一、常见误区

- TCP Keepalive 不是 HTTP Keep-Alive。
- TIME_WAIT 不是一定有害，它是正常协议状态。
- 禁用 Nagle 不是万能优化，要看业务是否对小包延迟敏感。
- 全连接队列满不一定是内核问题，也可能是应用 accept 或处理太慢。
- TCP 保证字节可靠有序，不保证应用消息边界。

## 十二、练习题

1. 滑动窗口解决什么问题？
2. 流量控制和拥塞控制分别保护谁？
3. Nagle 算法为什么可能增加小请求延迟？
4. TCP Keepalive、HTTP Keep-Alive、应用层心跳有什么区别？
5. 半连接队列和全连接队列分别处于三次握手的哪个阶段？

## 十三、验收标准

你能把 TCP 的窗口、队列、Keepalive、Nagle 与 Go 服务的超时、连接池、心跳、限流联系起来，并能用 `ss`、`sysctl`、压测工具观察连接状态变化。

