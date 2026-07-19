# 10. HTTP 反向代理完整落地：从目录到运行

本节目标：写一个最小可用的 HTTP 反向代理，完成单上游转发、多上游轮询、超时控制、请求日志和错误返回。

反向代理是理解 Nginx、API 网关、负载均衡和微服务入口的基础。Go 标准库已经提供 `httputil.ReverseProxy`，学习时应该先理解它，而不是一上来手写所有转发细节。

---

## 一、最终目标

项目目录：

```text
reverse-proxy-lab/
  go.mod
  cmd/backend/main.go
  cmd/proxy/main.go
```

运行后会有：

```text
backend A: 127.0.0.1:9001
backend B: 127.0.0.1:9002
proxy:     127.0.0.1:8080
```

访问代理：

```text
http://localhost:8080/api/hello
```

请求会轮流转发到 A 和 B。

---

## 二、创建项目

```powershell
mkdir reverse-proxy-lab
cd reverse-proxy-lab
go mod init example.com/reverse-proxy-lab
mkdir cmd
mkdir cmd\backend
mkdir cmd\proxy
```

这个项目特意不引入配置文件，先把核心转发链路跑通。

---

## 三、实现测试后端

创建：

```text
cmd/backend/main.go
```

写入：

```go
package main

import (
    "encoding/json"
    "flag"
    "log"
    "net/http"
    "time"
)

func main() {
    addr := flag.String("addr", ":9001", "listen address")
    name := flag.String("name", "backend-a", "backend name")
    delay := flag.Duration("delay", 0, "response delay")
    flag.Parse()

    mux := http.NewServeMux()
    mux.HandleFunc("GET /api/hello", func(w http.ResponseWriter, r *http.Request) {
        if *delay > 0 {
            time.Sleep(*delay)
        }
        w.Header().Set("Content-Type", "application/json")
        _ = json.NewEncoder(w).Encode(map[string]string{
            "backend": *name,
            "path":    r.URL.Path,
            "host":    r.Host,
        })
    })

    mux.HandleFunc("GET /healthz", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        _, _ = w.Write([]byte("ok"))
    })

    srv := &http.Server{
        Addr:              *addr,
        Handler:           mux,
        ReadHeaderTimeout: 3 * time.Second,
    }

    log.Printf("%s listening on %s", *name, *addr)
    log.Fatal(srv.ListenAndServe())
}
```

这个后端提供两个接口：

- `/api/hello` 用于观察代理转发到了哪个后端。
- `/healthz` 用于后续扩展健康检查。

`delay` 参数用于模拟慢后端。

---

## 四、实现轮询选择器

创建：

```text
cmd/proxy/main.go
```

先写基础结构：

```go
package main

import (
    "errors"
    "log"
    "net"
    "net/http"
    "net/http/httputil"
    "net/url"
    "sync/atomic"
    "time"
)

type upstreamPicker struct {
    urls []*url.URL
    next uint64
}

func newUpstreamPicker(rawURLs []string) (*upstreamPicker, error) {
    picker := &upstreamPicker{}
    for _, raw := range rawURLs {
        u, err := url.Parse(raw)
        if err != nil {
            return nil, err
        }
        picker.urls = append(picker.urls, u)
    }
    if len(picker.urls) == 0 {
        return nil, errors.New("no upstreams")
    }
    return picker, nil
}

func (p *upstreamPicker) pick() *url.URL {
    n := atomic.AddUint64(&p.next, 1)
    return p.urls[int(n-1)%len(p.urls)]
}
```

这里用 `atomic.AddUint64` 实现并发安全的轮询计数。

如果每个请求都读写普通整数，会有数据竞争。你可以用 `go test -race` 或压测工具暴露这类问题。

---

## 五、创建 ReverseProxy

继续写：

```go
func newProxy(picker *upstreamPicker) http.Handler {
    proxy := &httputil.ReverseProxy{
        Director: func(req *http.Request) {
            upstream := picker.pick()
            req.URL.Scheme = upstream.Scheme
            req.URL.Host = upstream.Host
            req.Host = upstream.Host

            req.Header.Set("X-Forwarded-Host", req.Host)
            req.Header.Set("X-Forwarded-Proto", "http")
        },
        Transport: &http.Transport{
            Proxy: http.ProxyFromEnvironment,
            DialContext: (&net.Dialer{
                Timeout:   2 * time.Second,
                KeepAlive: 30 * time.Second,
            }).DialContext,
            MaxIdleConns:          100,
            MaxIdleConnsPerHost:   20,
            IdleConnTimeout:       60 * time.Second,
            TLSHandshakeTimeout:   2 * time.Second,
            ResponseHeaderTimeout: 3 * time.Second,
        },
        ErrorHandler: func(w http.ResponseWriter, r *http.Request, err error) {
            log.Printf("proxy error method=%s path=%s err=%v", r.Method, r.URL.Path, err)
            http.Error(w, "bad gateway", http.StatusBadGateway)
        },
    }

    return accessLog(proxy)
}
```

几个关键字段：

- `Director` 在请求发往上游前修改请求。
- `Transport` 控制代理到上游的连接池、建连超时和响应头超时。
- `ResponseHeaderTimeout` 表示连接建立后，最多等待上游响应头多久。
- `ErrorHandler` 决定上游失败时返回什么。

注意 `X-Forwarded-*` 头是反向代理常见约定。真实生产中还需要处理客户端 IP、协议、链路追踪等信息。

---

## 六、增加访问日志

继续写：

```go
type statusWriter struct {
    http.ResponseWriter
    status int
}

func (w *statusWriter) WriteHeader(code int) {
    w.status = code
    w.ResponseWriter.WriteHeader(code)
}

func accessLog(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        sw := &statusWriter{ResponseWriter: w, status: http.StatusOK}
        start := time.Now()
        next.ServeHTTP(sw, r)
        log.Printf("method=%s path=%s status=%d cost=%s remote=%s",
            r.Method,
            r.URL.Path,
            sw.status,
            time.Since(start),
            r.RemoteAddr,
        )
    })
}
```

为什么要包装 `ResponseWriter`？

因为标准库没有直接提供“本次响应状态码”的读取方法。我们通过重写 `WriteHeader` 记录状态码。

---

## 七、代理 main 函数

继续写：

```go
func main() {
    picker, err := newUpstreamPicker([]string{
        "http://127.0.0.1:9001",
        "http://127.0.0.1:9002",
    })
    if err != nil {
        log.Fatal(err)
    }

    srv := &http.Server{
        Addr:              ":8080",
        Handler:           newProxy(picker),
        ReadHeaderTimeout: 3 * time.Second,
        ReadTimeout:       10 * time.Second,
        WriteTimeout:      10 * time.Second,
        IdleTimeout:       60 * time.Second,
    }

    log.Printf("proxy listening on %s", srv.Addr)
    log.Fatal(srv.ListenAndServe())
}
```

到这里，一个最小反向代理就完成了。

---

## 八、运行验证

终端 1 启动后端 A：

```powershell
go run ./cmd/backend -addr :9001 -name backend-a
```

终端 2 启动后端 B：

```powershell
go run ./cmd/backend -addr :9002 -name backend-b
```

终端 3 启动代理：

```powershell
go run ./cmd/proxy
```

终端 4 连续访问：

```powershell
curl.exe http://localhost:8080/api/hello
curl.exe http://localhost:8080/api/hello
curl.exe http://localhost:8080/api/hello
curl.exe http://localhost:8080/api/hello
```

你应该看到 `backend-a` 和 `backend-b` 轮流出现。

---

## 九、模拟 502

停止后端 A，只保留后端 B，然后继续访问代理。

因为轮询仍然会选到 A，所以部分请求会返回：

```http
502 Bad Gateway
```

代理日志会出现类似：

```text
proxy error method=GET path=/api/hello err=dial tcp 127.0.0.1:9001: connect: connection refused
```

这就是线上常见的 502 场景：

```text
代理能收到客户端请求，但代理连接不上上游。
```

---

## 十、模拟 504 类超时

启动一个慢后端：

```powershell
go run ./cmd/backend -addr :9001 -name slow-a -delay 5s
```

代理里的 `ResponseHeaderTimeout` 是 3 秒，所以访问时会等待一段时间后返回 `502 Bad Gateway`。

为什么不是标准 `504 Gateway Timeout`？

因为我们在 `ErrorHandler` 里统一写了 `502`。如果要更接近 Nginx 行为，可以根据错误类型区分：

```text
连接失败 -> 502
等待上游响应超时 -> 504
```

本节先保留简单版本，重点是理解“上游连接失败”和“上游响应太慢”的差异。

---

## 十一、常见问题

### 1. 反向代理为什么要改 `req.URL.Host`？

客户端请求代理时，目标 host 是 `localhost:8080`。

代理转发给上游时，必须把目标 host 改成 `127.0.0.1:9001` 或 `127.0.0.1:9002`，否则请求不会发到正确后端。

### 2. 为什么要配置代理自己的超时？

代理是网络链路中间层。如果没有超时，慢上游会占住代理连接，最终拖垮代理。

超时应该分层配置：

```text
客户端到代理。
代理到上游。
上游自己的处理。
```

### 3. 为什么轮询没有跳过宕机后端？

因为本节实现的是最小负载均衡。要跳过宕机后端，需要健康检查和摘除机制。

你可以在扩展练习里实现定时访问 `/healthz`，把失败的上游临时标记为不可用。

### 4. 这个代理和 Nginx 有什么关系？

它们解决的问题相似：

- 接收客户端请求。
- 选择上游。
- 转发请求。
- 返回响应。
- 记录日志。
- 处理上游错误。

Nginx 是成熟高性能产品，本节 Go 版本用于理解核心原理。

---

## 十二、扩展练习

继续完成：

1. 增加配置文件，允许配置上游列表。
2. 增加健康检查，失败 3 次后临时摘除上游。
3. 根据错误类型返回 `502` 或 `504`。
4. 增加 `X-Request-ID`。
5. 为不同路径配置不同 upstream。

---

## 十三、本节达标标准

学完本节后，你应该能够做到：

- 使用 `httputil.ReverseProxy` 写一个反向代理。
- 解释 `Director`、`Transport`、`ErrorHandler` 的职责。
- 实现并发安全的轮询负载均衡。
- 配置代理到上游的连接池和超时。
- 记录请求访问日志和状态码。
- 复现上游连接失败导致的 502。
- 复现上游响应慢导致的代理超时。
- 说明反向代理、负载均衡、API 网关之间的关系。

