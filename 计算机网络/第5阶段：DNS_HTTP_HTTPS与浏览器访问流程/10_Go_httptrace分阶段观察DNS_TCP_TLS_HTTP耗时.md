# 10. Go httptrace：分阶段观察 DNS、TCP、TLS、HTTP 耗时

本节目标：使用 Go 标准库 `net/http/httptrace` 观察一次 HTTP 请求中 DNS、TCP 连接、TLS 握手、等待响应头等阶段的耗时。学完后你能把“接口慢”拆成更细的网络阶段，而不是只看总耗时。

---

## 一、为什么要学 httptrace

线上经常出现这种问题：

```text
用户说接口很慢。
服务端日志显示 handler 很快。
Go 客户端日志只记录了总耗时。
```

这时你需要知道慢在哪里：

```text
DNS 慢？
TCP 连接慢？
TLS 握手慢？
等待响应头慢？
读取 body 慢？
```

`httptrace` 可以帮助你在 Go 客户端侧观察这些阶段。

---

## 二、创建项目

```bash
mkdir httptrace-lab
cd httptrace-lab
go mod init httptrace-lab
```

创建 `main.go`。

---

## 三、完整代码

```go
package main

import (
    "context"
    "crypto/tls"
    "fmt"
    "io"
    "net/http"
    "net/http/httptrace"
    "time"
)

func main() {
    url := "https://example.com"

    var (
        start              = time.Now()
        dnsStart           time.Time
        connectStart       time.Time
        tlsStart           time.Time
        wroteRequest       time.Time
        gotFirstResponse   time.Time
    )

    trace := &httptrace.ClientTrace{
        DNSStart: func(info httptrace.DNSStartInfo) {
            dnsStart = time.Now()
            fmt.Println("DNSStart:", info.Host)
        },
        DNSDone: func(info httptrace.DNSDoneInfo) {
            fmt.Println("DNSDone:", time.Since(dnsStart), info.Addrs, "err=", info.Err)
        },
        ConnectStart: func(network, addr string) {
            connectStart = time.Now()
            fmt.Println("ConnectStart:", network, addr)
        },
        ConnectDone: func(network, addr string, err error) {
            fmt.Println("ConnectDone:", time.Since(connectStart), "err=", err)
        },
        TLSHandshakeStart: func() {
            tlsStart = time.Now()
            fmt.Println("TLSHandshakeStart")
        },
        TLSHandshakeDone: func(state tls.ConnectionState, err error) {
            fmt.Println("TLSHandshakeDone:", time.Since(tlsStart), "version=", state.Version, "err=", err)
        },
        WroteRequest: func(info httptrace.WroteRequestInfo) {
            wroteRequest = time.Now()
            fmt.Println("WroteRequest:", "err=", info.Err)
        },
        GotFirstResponseByte: func() {
            gotFirstResponse = time.Now()
            fmt.Println("GotFirstResponseByte:", time.Since(wroteRequest))
        },
    }

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    ctx = httptrace.WithClientTrace(ctx, trace)

    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        panic(err)
    }

    client := &http.Client{
        Timeout: 10 * time.Second,
        Transport: &http.Transport{
            MaxIdleConns:          100,
            MaxIdleConnsPerHost:   20,
            IdleConnTimeout:       90 * time.Second,
            TLSHandshakeTimeout:   3 * time.Second,
            ResponseHeaderTimeout: 5 * time.Second,
        },
    }

    resp, err := client.Do(req)
    if err != nil {
        panic(err)
    }
    defer resp.Body.Close()

    n, err := io.Copy(io.Discard, resp.Body)
    if err != nil {
        panic(err)
    }

    fmt.Println("Status:", resp.Status)
    fmt.Println("Body bytes:", n)
    fmt.Println("Total:", time.Since(start))
}
```

---

## 四、运行

```bash
go run .
```

你可能看到类似：

```text
DNSStart: example.com
DNSDone: 15ms [...]
ConnectStart: tcp 93.184.216.34:443
ConnectDone: 80ms err=<nil>
TLSHandshakeStart
TLSHandshakeDone: 120ms version=772 err=<nil>
WroteRequest: err=<nil>
GotFirstResponseByte: 60ms
Status: 200 OK
Body bytes: 1256
Total: 300ms
```

不同网络环境输出会不同。

---

## 五、如何解读

### DNS 慢

如果 DNSDone 耗时很长：

```text
检查 /etc/resolv.conf。
换 DNS 服务器测试。
检查容器或 Kubernetes DNS。
```

### Connect 慢

如果 ConnectDone 很慢：

```text
检查路由、防火墙、目标端口、跨地域网络。
```

### TLS 慢

如果 TLSHandshakeDone 很慢：

```text
检查证书、TLS 配置、网络 RTT、代理。
```

### GotFirstResponseByte 慢

如果写完请求后很久才收到第一个响应字节：

```text
通常是服务端处理慢、网关排队、上游依赖慢。
```

### 读取 body 慢

如果首字节快但总耗时长：

```text
可能是响应体大、带宽慢、客户端读取慢。
```

---

## 六、连接复用对 trace 的影响

如果 HTTP 连接被复用，可能不会触发 DNS、Connect、TLS 阶段。

你可以连续请求两次观察：

```go
for i := 0; i < 2; i++ {
    // 执行请求
}
```

第二次请求可能直接复用已有连接。

这正好说明连接池的价值：

```text
减少 DNS、TCP、TLS 开销。
```

---

## 七、常见问题

### 1. httptrace 能替代服务端 tracing 吗？

不能。httptrace 主要观察客户端侧网络阶段，服务端内部耗时仍需要日志、metrics、tracing。

### 2. GotFirstResponseByte 慢一定是服务端业务慢吗？

不一定。也可能是网关排队、上游网络慢、服务端写响应头慢。

### 3. 为什么第二次请求没有 DNS 和 TLS 输出？

可能复用了连接。复用连接时不需要重新 DNS、TCP、TLS。

### 4. Client.Timeout 和 ResponseHeaderTimeout 有什么区别？

`Client.Timeout` 是整体请求超时，`ResponseHeaderTimeout` 是等待响应头的超时。

---

## 八、本节达标标准

学完本节后，你应该能够做到：

- 使用 `httptrace.ClientTrace` 观察 DNS、TCP、TLS、首字节时间。
- 解释每个阶段慢可能对应什么问题。
- 理解连接复用为什么会让部分 trace 回调不触发。
- 把“接口慢”拆成可观测的多个阶段。

