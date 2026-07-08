# 5. 启动一个 Go HTTP 服务并完成端到端访问

本节目标：编写并启动一个最小 Go HTTP 服务，使用 curl 访问它，使用 ss 查看监听端口，使用 tcpdump 或 Wireshark 观察请求，并理解一次本地 HTTP 请求的完整链路。

这是本阶段最重要的一节。前面学的 IP、端口、Docker、抓包，最终都要落到一个问题上：

```text
我写的 Go 服务，别人到底是怎么访问到的？
```

---

## 一、创建项目

创建目录：

```bash
mkdir hello-http
cd hello-http
go mod init hello-http
```

创建 `main.go`：

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
    "time"
)

type Response struct {
    Message string `json:"message"`
    Path    string `json:"path"`
}

func main() {
    mux := http.NewServeMux()

    mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusNoContent)
    })

    mux.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(Response{
            Message: "hello network",
            Path:    r.URL.Path,
        })
    })

    srv := &http.Server{
        Addr:              ":8080",
        Handler:           mux,
        ReadHeaderTimeout: 2 * time.Second,
        ReadTimeout:       5 * time.Second,
        WriteTimeout:      10 * time.Second,
        IdleTimeout:       60 * time.Second,
    }

    log.Println("listen on http://127.0.0.1:8080")
    log.Fatal(srv.ListenAndServe())
}
```

---

## 二、为什么不用最简单的 ListenAndServe

你可能见过：

```go
http.ListenAndServe(":8080", nil)
```

这确实能启动服务，但生产中不推荐这样写，因为它没有显式配置超时。

本节使用：

```go
srv := &http.Server{
    ReadHeaderTimeout: 2 * time.Second,
    ReadTimeout:       5 * time.Second,
    WriteTimeout:      10 * time.Second,
    IdleTimeout:       60 * time.Second,
}
```

原因是：

- `ReadHeaderTimeout` 防止客户端慢慢发送 Header。
- `ReadTimeout` 限制读取请求的总时间。
- `WriteTimeout` 限制写响应的时间。
- `IdleTimeout` 控制长连接空闲时间。

你现在不必完全掌握每个超时的细节，但要先养成习惯：后端网络服务要考虑超时。

---

## 三、启动服务

运行：

```bash
go run .
```

看到：

```text
listen on http://127.0.0.1:8080
```

说明程序已经启动。

注意：日志里写 `127.0.0.1` 只是提示你可以这样访问。实际 `Addr: ":8080"` 通常监听所有网卡。

---

## 四、查看端口监听

另一个终端执行：

```bash
ss -lntp | grep 8080
```

你可能看到：

```text
LISTEN 0 4096 *:8080
```

或：

```text
LISTEN 0 4096 0.0.0.0:8080
```

这说明服务正在监听 `8080` 端口。

如果看不到，说明服务可能没有启动成功，或者端口不是 `8080`。

---

## 五、使用 curl 访问服务

访问健康检查：

```bash
curl -i http://127.0.0.1:8080/health
```

你应该看到：

```http
HTTP/1.1 204 No Content
```

访问业务接口：

```bash
curl -v http://127.0.0.1:8080/hello
```

关注 `curl -v` 输出中的这些部分：

```text
Trying 127.0.0.1:8080...
Connected to 127.0.0.1
> GET /hello HTTP/1.1
< HTTP/1.1 200 OK
```

它说明：

- curl 尝试连接 `127.0.0.1:8080`。
- TCP 连接成功。
- 发送了 HTTP GET 请求。
- 收到了 HTTP 200 响应。

---

## 六、观察连接状态

请求时执行：

```bash
ss -antp | grep 8080
```

你可能看到：

```text
ESTAB
TIME-WAIT
```

含义先简单理解：

- `ESTAB`：连接已建立。
- `TIME-WAIT`：连接关闭后保留一段时间。

后面 TCP 阶段会详细解释这些状态。

---

## 七、抓包观察请求

使用 tcpdump：

```bash
sudo tcpdump -i any tcp port 8080 -nn
```

另一个终端请求：

```bash
curl http://127.0.0.1:8080/hello
```

你会看到 TCP 报文。

保存抓包文件：

```bash
sudo tcpdump -i any tcp port 8080 -c 20 -w hello-http.pcap
```

再请求一次：

```bash
curl http://127.0.0.1:8080/hello
```

用 Wireshark 打开 `hello-http.pcap`，过滤：

```text
http || tcp
```

观察：

- TCP 建立连接。
- HTTP GET 请求。
- HTTP 响应。
- TCP 连接关闭。

---

## 八、故意制造错误：访问不存在的路径

请求：

```bash
curl -i http://127.0.0.1:8080/not-found
```

你会看到：

```http
HTTP/1.1 404 Not Found
```

原因是我们只注册了：

```text
/health
/hello
```

没有注册 `/not-found`。

这说明：

```text
端口通了，不代表业务路径存在。
```

---

## 九、故意制造错误：访问错误端口

请求：

```bash
curl -v http://127.0.0.1:9090/hello
```

如果 `9090` 没有服务监听，通常会看到：

```text
Connection refused
```

这表示：

```text
目标主机可达，但目标端口没有程序监听。
```

这和超时不同。超时通常更像：

```text
包被丢弃、防火墙拦截、路由不通、对方没有响应。
```

---

## 十、改成只监听 127.0.0.1

把代码改成：

```go
Addr: "127.0.0.1:8080",
```

重新启动。

查看：

```bash
ss -lntp | grep 8080
```

你应该看到监听地址变成 `127.0.0.1:8080`。

这时本机仍然可以访问：

```bash
curl http://127.0.0.1:8080/hello
```

但其他机器通常无法通过你的局域网 IP 访问它。

这就是很多部署问题的根源。

---

## 十一、一次请求背后的完整链路

当你执行：

```bash
curl http://127.0.0.1:8080/hello
```

大致发生：

```text
curl 解析 URL。
curl 连接 127.0.0.1:8080。
操作系统建立 TCP 连接。
Go HTTP Server 接收连接。
Go 路由匹配 /hello。
handler 写 JSON 响应。
curl 读取响应并打印。
连接关闭或进入复用。
```

如果换成域名和 HTTPS，还会多出：

```text
DNS 解析。
TLS 握手。
证书验证。
```

---

## 十二、常见问题

### 1. 启动时报端口被占用

查看占用：

```bash
ss -lntp | grep 8080
```

换端口或停止占用进程。

### 2. curl 返回 Connection refused

优先检查：

- 服务是否启动。
- 端口是否正确。
- 监听地址是否正确。

### 3. curl 一直卡住

可能原因：

- 服务端没有返回。
- 防火墙丢包。
- 请求打到了错误地址。
- 网络路径不通。

可以用：

```bash
curl -v --max-time 3 http://目标地址
```

限制最大等待时间。

### 4. 为什么服务端没有日志？

可能请求根本没到服务端，例如：

- DNS 解析失败。
- TCP 连接失败。
- 端口错误。
- 防火墙拦截。

后续排障阶段会专门训练这个问题。

---

## 十三、本节达标标准

学完本节后，你应该能够做到：

- 从空目录创建并运行一个 Go HTTP 服务。
- 使用 `curl -i` 和 `curl -v` 观察响应和连接过程。
- 使用 `ss -lntp` 确认端口监听。
- 使用 tcpdump 或 Wireshark 抓到访问服务的报文。
- 解释 `Connection refused` 和 `404 Not Found` 的区别。
- 解释为什么监听 `127.0.0.1` 会影响外部访问。

完成这些之后，第 1 阶段就达标了。下一阶段可以开始系统学习网络分层、MAC、IP、端口、ARP 和 ICMP。

