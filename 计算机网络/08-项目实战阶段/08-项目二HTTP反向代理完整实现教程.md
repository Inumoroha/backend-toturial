# 08-项目二：HTTP 反向代理完整实现教程

## 学习目标

从 0 到 1 实现一个简易 HTTP 反向代理。完成后你会理解请求转发、Header 处理、连接池、超时、错误码、请求日志和简单负载均衡。

## 一、最终效果

启动两个后端：

```bash
APP_NAME=backend-1 PORT=8081 go run ./cmd/backend
APP_NAME=backend-2 PORT=8082 go run ./cmd/backend
```

启动代理：

```bash
go run ./cmd/proxy
```

请求代理：

```bash
curl -v http://127.0.0.1:8080/hello
```

多请求几次，可以看到请求轮询到不同后端。

## 二、创建项目

```bash
mkdir http-reverse-proxy
cd http-reverse-proxy
go mod init http-reverse-proxy
mkdir -p cmd/backend cmd/proxy internal/proxy
```

目录：

```text
http-reverse-proxy/
  go.mod
  cmd/
    backend/
      main.go
    proxy/
      main.go
  internal/
    proxy/
      balancer.go
      logging.go
```

## 三、实现测试后端

创建 `cmd/backend/main.go`：

```go
package main

import (
    "fmt"
    "log"
    "net/http"
    "os"
    "time"
)

func main() {
    name := os.Getenv("APP_NAME")
    if name == "" {
        name = "backend"
    }

    port := os.Getenv("PORT")
    if port == "" {
        port = "8081"
    }

    mux := http.NewServeMux()
    mux.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "hello from %s\n", name)
    })
    mux.HandleFunc("/slow", func(w http.ResponseWriter, r *http.Request) {
        time.Sleep(3 * time.Second)
        fmt.Fprintf(w, "slow response from %s\n", name)
    })
    mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusNoContent)
    })

    srv := &http.Server{
        Addr:              ":" + port,
        Handler:           mux,
        ReadHeaderTimeout: 2 * time.Second,
        ReadTimeout:       5 * time.Second,
        WriteTimeout:      5 * time.Second,
        IdleTimeout:       60 * time.Second,
    }

    log.Println(name, "listen on", srv.Addr)
    log.Fatal(srv.ListenAndServe())
}
```

## 四、实现轮询负载均衡器

创建 `internal/proxy/balancer.go`：

```go
package proxy

import (
    "net/url"
    "sync"
)

type Balancer struct {
    mu      sync.Mutex
    targets []*url.URL
    next    int
}

func NewBalancer(rawTargets []string) (*Balancer, error) {
    targets := make([]*url.URL, 0, len(rawTargets))
    for _, raw := range rawTargets {
        u, err := url.Parse(raw)
        if err != nil {
            return nil, err
        }
        targets = append(targets, u)
    }

    return &Balancer{targets: targets}, nil
}

func (b *Balancer) Pick() *url.URL {
    b.mu.Lock()
    defer b.mu.Unlock()

    target := b.targets[b.next%len(b.targets)]
    b.next++
    return target
}
```

## 五、记录状态码的 ResponseWriter

创建 `internal/proxy/logging.go`：

```go
package proxy

import (
    "log"
    "net/http"
    "time"
)

type statusWriter struct {
    http.ResponseWriter
    status int
}

func (w *statusWriter) WriteHeader(status int) {
    w.status = status
    w.ResponseWriter.WriteHeader(status)
}

func Logging(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        sw := &statusWriter{ResponseWriter: w, status: http.StatusOK}

        next.ServeHTTP(sw, r)

        log.Printf(
            "method=%s path=%s status=%d cost=%s remote=%s",
            r.Method,
            r.URL.Path,
            sw.status,
            time.Since(start),
            r.RemoteAddr,
        )
    })
}
```

## 六、实现代理入口

创建 `cmd/proxy/main.go`：

```go
package main

import (
    "context"
    "errors"
    "log"
    "net"
    "net/http"
    "net/http/httputil"
    "net/url"
    "time"

    proxykit "http-reverse-proxy/internal/proxy"
)

func main() {
    balancer, err := proxykit.NewBalancer([]string{
        "http://127.0.0.1:8081",
        "http://127.0.0.1:8082",
    })
    if err != nil {
        log.Fatal(err)
    }

    reverseProxy := &httputil.ReverseProxy{
        Director: func(r *http.Request) {
            target := balancer.Pick()
            rewriteRequest(r, target)
        },
        Transport: &http.Transport{
            Proxy: http.ProxyFromEnvironment,
            DialContext: (&net.Dialer{
                Timeout:   1 * time.Second,
                KeepAlive: 30 * time.Second,
            }).DialContext,
            MaxIdleConns:          100,
            MaxIdleConnsPerHost:   20,
            MaxConnsPerHost:       100,
            IdleConnTimeout:       90 * time.Second,
            TLSHandshakeTimeout:   3 * time.Second,
            ResponseHeaderTimeout: 1 * time.Second,
        },
        ErrorHandler: func(w http.ResponseWriter, r *http.Request, err error) {
            log.Printf("proxy error path=%s err=%v", r.URL.Path, err)
            if errors.Is(err, context.DeadlineExceeded) {
                http.Error(w, "gateway timeout", http.StatusGatewayTimeout)
                return
            }
            http.Error(w, "bad gateway", http.StatusBadGateway)
        },
    }

    srv := &http.Server{
        Addr:              ":8080",
        Handler:           proxykit.Logging(reverseProxy),
        ReadHeaderTimeout: 2 * time.Second,
        ReadTimeout:       5 * time.Second,
        WriteTimeout:      5 * time.Second,
        IdleTimeout:       60 * time.Second,
    }

    log.Println("proxy listen on :8080")
    log.Fatal(srv.ListenAndServe())
}

func rewriteRequest(r *http.Request, target *url.URL) {
    originalHost := r.Host

    r.URL.Scheme = target.Scheme
    r.URL.Host = target.Host
    r.Host = target.Host

    r.Header.Set("X-Forwarded-Host", originalHost)
    r.Header.Set("X-Forwarded-Proto", "http")
}
```

## 七、运行验证

终端 1：

```bash
APP_NAME=backend-1 PORT=8081 go run ./cmd/backend
```

终端 2：

```bash
APP_NAME=backend-2 PORT=8082 go run ./cmd/backend
```

终端 3：

```bash
go run ./cmd/proxy
```

终端 4：

```bash
curl http://127.0.0.1:8080/hello
curl http://127.0.0.1:8080/hello
curl http://127.0.0.1:8080/hello
```

你应该能看到响应来自不同后端。

## 八、测试超时

请求慢接口：

```bash
curl -v http://127.0.0.1:8080/slow
```

后端会 sleep 3 秒，但代理 `ResponseHeaderTimeout` 是 1 秒，所以应该返回 502 或 504。由于底层错误类型不一定总是 `context.DeadlineExceeded`，你可以在日志中观察具体错误，再细化 ErrorHandler。

这也是工程实践的重要点：不要只写理想代码，要观察真实错误。

## 九、观察连接池

```bash
ss -antp | grep 808
```

多次请求后，你可能会看到代理到后端的连接保持一段时间，这是 HTTP Keep-Alive 和 Transport 连接池在工作。

## 十、抓包观察

抓代理入口：

```bash
sudo tcpdump -i any tcp port 8080 -nn
```

抓代理到后端：

```bash
sudo tcpdump -i any 'tcp port 8081 or tcp port 8082' -nn
```

你可以对比：

- 客户端到代理的连接。
- 代理到后端的连接。
- 两段连接不是同一个 TCP 连接。

## 十一、当前版本的不足

这个版本已经能工作，但还不是生产网关：

- 没有健康检查，坏后端仍可能被选中。
- 没有配置文件。
- 没有限流。
- 没有熔断。
- 没有 request id。
- ErrorHandler 对超时错误识别还可以更细。

这些不足正好对应迷你 API 网关项目。

## 十二、练习任务

1. 增加 `/health`，返回代理自身状态。
2. 增加配置文件读取后端列表。
3. 给每个请求生成 `X-Request-Id`。
4. 健康检查失败时暂时摘除后端。
5. 增加简单限流，超过限制返回 429。

## 十三、验收标准

你能从空目录创建项目，启动两个后端和一个代理，请求代理后轮询访问后端，并能解释代理的超时、连接池、错误处理和日志记录。
