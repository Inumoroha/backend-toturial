# 8. TCP 抓包实验：握手、挥手、状态变化

本节目标：通过可复现实验观察 TCP 三次握手、四次挥手、TIME_WAIT、连接拒绝和连接超时。学完后，你不只是能背 TCP 状态，而是能用 `tcpdump`、`ss`、`curl` 证明连接发生了什么。

---

## 一、实验准备

创建目录：

```bash
mkdir tcp-state-lab
cd tcp-state-lab
go mod init tcp-state-lab
```

创建 `main.go`：

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "time"
)

func main() {
    mux := http.NewServeMux()

    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "ok")
    })

    mux.HandleFunc("/slow", func(w http.ResponseWriter, r *http.Request) {
        time.Sleep(5 * time.Second)
        fmt.Fprintln(w, "slow ok")
    })

    srv := &http.Server{
        Addr:              ":8080",
        Handler:           mux,
        ReadHeaderTimeout: 2 * time.Second,
        ReadTimeout:       10 * time.Second,
        WriteTimeout:      10 * time.Second,
        IdleTimeout:       30 * time.Second,
    }

    log.Println("listen on :8080")
    log.Fatal(srv.ListenAndServe())
}
```

启动：

```bash
go run .
```

另开终端确认监听：

```bash
ss -lntp | grep 8080
```

你应该看到 `:8080` 正在监听。

---

## 二、实验一：抓三次握手

终端 A 抓包：

```bash
sudo tcpdump -i any tcp port 8080 -nn
```

终端 B 请求：

```bash
curl http://127.0.0.1:8080/
```

你可能看到类似输出：

```text
127.0.0.1.51432 > 127.0.0.1.8080: Flags [S]
127.0.0.1.8080 > 127.0.0.1.51432: Flags [S.]
127.0.0.1.51432 > 127.0.0.1.8080: Flags [.]
```

逐行解释：

- 第一行 `[S]`：客户端发送 SYN，请求建立连接。
- 第二行 `[S.]`：服务端返回 SYN + ACK。
- 第三行 `[.]`：客户端返回 ACK，连接建立完成。

这里的 `51432` 是客户端临时端口，`8080` 是服务端监听端口。

---

## 三、实验二：观察 ESTABLISHED

请求慢接口：

```bash
curl http://127.0.0.1:8080/slow
```

在请求未结束时，另一个终端执行：

```bash
ss -antp | grep 8080
```

你可能看到：

```text
ESTAB 0 0 127.0.0.1:8080 127.0.0.1:51440
ESTAB 0 0 127.0.0.1:51440 127.0.0.1:8080
```

解释：

- `ESTAB` 表示 TCP 连接已建立。
- 本机回环访问时，客户端和服务端都在同一台机器，所以可能看到两条方向不同的记录。

---

## 四、实验三：观察 TIME_WAIT

连续请求：

```bash
for i in {1..50}; do curl -s http://127.0.0.1:8080/ > /dev/null; done
```

查看 TIME_WAIT：

```bash
ss -ant state time-wait | grep 8080 | head
```

你可能看到一些 TIME_WAIT 连接。

解释：

```text
主动关闭连接的一方通常进入 TIME_WAIT。
```

单独使用 curl 反复请求时，经常产生短连接。真实 Go 服务中，如果 HTTP Client 没有复用连接，也可能造成大量 TIME_WAIT。

---

## 五、实验四：连接被拒绝

停止 Go 服务：

```text
Ctrl+C
```

再次请求：

```bash
curl -v http://127.0.0.1:8080/
```

常见输出：

```text
Connection refused
```

解释：

```text
目标主机可达，但目标端口没有服务监听，或者被主动拒绝。
```

用命令证明：

```bash
ss -lntp | grep 8080
```

如果没有输出，说明端口确实没有监听。

---

## 六、实验五：连接超时与防火墙丢弃的区别

连接超时通常需要访问一个不会响应的地址或被防火墙丢弃的端口。

示例：

```bash
curl -v --connect-timeout 3 http://10.255.255.1:8080/
```

可能输出：

```text
Connection timed out
```

解释：

- `refused` 通常是收到拒绝。
- `timeout` 通常是没有收到响应。

真实环境中，timeout 常见原因：

- 防火墙丢弃。
- 安全组未放行。
- 路由不通。
- 目标机器无响应。

---

## 七、实验六：保存 pcap 用 Wireshark 分析

抓包保存：

```bash
sudo tcpdump -i any tcp port 8080 -c 50 -w tcp-8080.pcap
```

另一个终端请求：

```bash
curl http://127.0.0.1:8080/
```

用 Wireshark 打开 `tcp-8080.pcap`。

显示过滤器：

```text
tcp.port == 8080
```

重点观察字段：

- `SYN`
- `ACK`
- `FIN`
- `RST`
- `Seq`
- `Ack`
- `Len`

初学阶段不要求你记住每个字段，但要能找到握手和关闭过程。

---

## 八、常见问题

### 1. 为什么本机抓包会看到两条 ESTABLISHED？

因为客户端和服务端都在同一台机器，本机同时看到连接的两个方向。

### 2. 为什么 curl 请求后不一定马上看到 TIME_WAIT？

连接可能被复用，也可能状态很快变化。不同系统、不同 curl 版本和连接策略会影响观察结果。

### 3. Connection refused 是不是网络不通？

不一定。它通常说明网络到了目标主机，但端口没有服务监听。

### 4. tcpdump 里 `[R]` 是什么？

`[R]` 表示 RST，连接被重置。可能由服务端、客户端或中间设备发出。

---

## 九、本节达标标准

学完本节后，你应该能够做到：

- 用 tcpdump 抓到 TCP 三次握手。
- 用 ss 观察 ESTABLISHED 和 TIME_WAIT。
- 区分 connection refused 和 timeout。
- 保存 pcap 并用 Wireshark 查看 TCP Flags。
- 用证据解释连接建立、请求、关闭的过程。

