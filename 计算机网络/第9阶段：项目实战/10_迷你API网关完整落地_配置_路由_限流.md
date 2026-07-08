# 10. 迷你 API 网关完整落地：配置、路由、限流与健康检查

本节目标：在反向代理项目的基础上，继续实现一个迷你 API 网关，支持按路径匹配上游、简单 Token 鉴权、IP 限流、健康检查和统一日志。

API 网关不是一个神秘组件。它本质上是在反向代理前后增加一组通用能力，让业务服务少重复写横切逻辑。

---

## 一、最终目标

项目目录：

```text
mini-gateway/
  go.mod
  cmd/backend/main.go
  cmd/gateway/main.go
```

网关能力：

```text
/users/*    -> user-service
/orders/*   -> order-service
Authorization: Bearer dev-token
每个客户端 IP 每秒最多 5 个请求
GET /healthz 返回网关健康状态
```

这不是生产级网关，但已经覆盖 API 网关的核心工作流：

```text
请求进入 -> 访问日志 -> 鉴权 -> 限流 -> 路由匹配 -> 转发上游 -> 错误处理
```

---

## 二、创建项目

```powershell
mkdir mini-gateway
cd mini-gateway
go mod init example.com/mini-gateway
mkdir cmd
mkdir cmd\backend
mkdir cmd\gateway
```

本节继续只用标准库，便于看清楚每个环节。

---

## 三、实现测试业务服务

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
    addr := flag.String("addr", ":9101", "listen address")
    service := flag.String("service", "user-service", "service name")
    flag.Parse()

    mux := http.NewServeMux()
    mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        _ = json.NewEncoder(w).Encode(map[string]string{
            "service": *service,
            "method":  r.Method,
            "path":    r.URL.Path,
        })
    })

    srv := &http.Server{
        Addr:              *addr,
        Handler:           mux,
        ReadHeaderTimeout: 3 * time.Second,
    }

    log.Printf("%s listening on %s", *service, *addr)
    log.Fatal(srv.ListenAndServe())
}
```

这个后端会把收到的路径原样返回，方便观察网关路由是否正确。

---

## 四、定义路由配置

创建：

```text
cmd/gateway/main.go
```

先写配置结构：

```go
package main

import (
    "log"
    "net"
    "net/http"
    "net/http/httputil"
    "net/url"
    "strings"
    "sync"
    "time"
)

type route struct {
    Prefix   string
    Upstream *url.URL
}

func mustURL(raw string) *url.URL {
    u, err := url.Parse(raw)
    if err != nil {
        log.Fatal(err)
    }
    return u
}

var routes = []route{
    {Prefix: "/users/", Upstream: mustURL("http://127.0.0.1:9101")},
    {Prefix: "/orders/", Upstream: mustURL("http://127.0.0.1:9102")},
}
```

真实项目会从 YAML、JSON、数据库或配置中心读取路由。本节先写死在代码里，降低干扰。

---

## 五、实现路由匹配

继续写：

```go
func matchRoute(path string) (*url.URL, bool) {
    for _, rt := range routes {
        if strings.HasPrefix(path, rt.Prefix) {
            return rt.Upstream, true
        }
    }
    return nil, false
}
```

这里使用前缀匹配：

```text
/users/123       匹配 /users/
/orders/2026001  匹配 /orders/
/products/1      不匹配
```

注意前缀里保留结尾 `/`，可以减少误匹配。例如 `/users2` 不应该匹配 `/users/`。

---

## 六、实现 Token 鉴权中间件

继续写：

```go
func auth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.URL.Path == "/healthz" {
            next.ServeHTTP(w, r)
            return
        }

        token := r.Header.Get("Authorization")
        if token != "Bearer dev-token" {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

本节使用固定 token 是为了突出网关流程。真实系统会对接 JWT、公钥验签、OAuth2、内部服务身份等机制。

鉴权放在转发之前，目的是避免非法请求打到业务服务。

---

## 七、实现简单 IP 限流

继续写：

```go
type limiter struct {
    mu      sync.Mutex
    buckets map[string]*bucket
    limit   int
}

type bucket struct {
    windowStart time.Time
    count       int
}

func newLimiter(limit int) *limiter {
    return &limiter{
        buckets: make(map[string]*bucket),
        limit:   limit,
    }
}

func (l *limiter) allow(key string) bool {
    l.mu.Lock()
    defer l.mu.Unlock()

    now := time.Now()
    b := l.buckets[key]
    if b == nil || now.Sub(b.windowStart) >= time.Second {
        l.buckets[key] = &bucket{windowStart: now, count: 1}
        return true
    }

    if b.count >= l.limit {
        return false
    }

    b.count++
    return true
}

func rateLimit(l *limiter, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        host, _, err := net.SplitHostPort(r.RemoteAddr)
        if err != nil {
            host = r.RemoteAddr
        }

        if !l.allow(host) {
            http.Error(w, "too many requests", http.StatusTooManyRequests)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

这是固定窗口限流：

```text
同一个 IP 在 1 秒窗口内最多 N 次。
```

它简单、容易理解，但不是最平滑的算法。生产中更常见的是令牌桶、漏桶、滑动窗口，或者直接使用 Redis/Envoy/Nginx/APISIX 等组件能力。

---

## 八、实现网关代理

继续写：

```go
func gatewayProxy() http.Handler {
    proxy := &httputil.ReverseProxy{
        Director: func(req *http.Request) {
            upstream, ok := matchRoute(req.URL.Path)
            if !ok {
                return
            }
            req.URL.Scheme = upstream.Scheme
            req.URL.Host = upstream.Host
            req.Host = upstream.Host
            req.Header.Set("X-Gateway", "mini-gateway")
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
            ResponseHeaderTimeout: 3 * time.Second,
        },
        ErrorHandler: func(w http.ResponseWriter, r *http.Request, err error) {
            log.Printf("gateway proxy error path=%s err=%v", r.URL.Path, err)
            http.Error(w, "bad gateway", http.StatusBadGateway)
        },
    }

    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if r.URL.Path == "/healthz" {
            w.WriteHeader(http.StatusOK)
            _, _ = w.Write([]byte("ok"))
            return
        }

        if _, ok := matchRoute(r.URL.Path); !ok {
            http.Error(w, "route not found", http.StatusNotFound)
            return
        }

        proxy.ServeHTTP(w, r)
    })
}
```

这里有一个小设计：

- 外层 handler 先判断路径是否匹配。
- 匹配后再交给 `ReverseProxy`。

这样未知路径能明确返回 `404 route not found`，而不是产生难懂的代理错误。

---

## 九、增加访问日志

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

网关层日志非常关键，因为它是客户端和服务端之间的边界。

当业务服务说“没收到请求”时，你可以先看网关日志：

```text
请求有没有进网关？
鉴权有没有通过？
是否被限流？
路由是否匹配？
转发到上游是否失败？
```

---

## 十、组合 main 函数

继续写：

```go
func main() {
    l := newLimiter(5)

    handler := gatewayProxy()
    handler = rateLimit(l, handler)
    handler = auth(handler)
    handler = accessLog(handler)

    srv := &http.Server{
        Addr:              ":8080",
        Handler:           handler,
        ReadHeaderTimeout: 3 * time.Second,
        ReadTimeout:       10 * time.Second,
        WriteTimeout:      10 * time.Second,
        IdleTimeout:       60 * time.Second,
    }

    log.Printf("mini gateway listening on %s", srv.Addr)
    log.Fatal(srv.ListenAndServe())
}
```

中间件顺序要看清楚：

```text
accessLog -> auth -> rateLimit -> gatewayProxy
```

也就是说，所有请求都会记录日志；非法 token 会在鉴权阶段被挡住；通过鉴权后再限流；最后才转发。

---

## 十一、运行验证

终端 1 启动用户服务：

```powershell
go run ./cmd/backend -addr :9101 -service user-service
```

终端 2 启动订单服务：

```powershell
go run ./cmd/backend -addr :9102 -service order-service
```

终端 3 启动网关：

```powershell
go run ./cmd/gateway
```

健康检查：

```powershell
curl.exe -i http://localhost:8080/healthz
```

未带 token：

```powershell
curl.exe -i http://localhost:8080/users/1
```

预期：

```http
401 Unauthorized
```

带 token 访问用户服务：

```powershell
curl.exe -i http://localhost:8080/users/1 -H "Authorization: Bearer dev-token"
```

带 token 访问订单服务：

```powershell
curl.exe -i http://localhost:8080/orders/1001 -H "Authorization: Bearer dev-token"
```

未知路径：

```powershell
curl.exe -i http://localhost:8080/products/1 -H "Authorization: Bearer dev-token"
```

预期：

```http
404 Not Found
```

---

## 十二、验证限流

快速执行：

```powershell
1..10 | ForEach-Object {
  curl.exe -s -o NUL -w "%{http_code}`n" http://localhost:8080/users/1 -H "Authorization: Bearer dev-token"
}
```

你应该看到前几个是：

```text
200
200
200
200
200
```

后面开始出现：

```text
429
```

这说明网关已经在业务服务前面挡住了过量请求。

---

## 十三、常见问题

### 1. 鉴权和限流谁先执行？

取决于你的目标。

本节先鉴权再限流，表示只有合法用户才占用限流桶。如果你想防止非法请求暴力攻击鉴权逻辑，也可以先按 IP 做一层粗限流，再鉴权。

### 2. 为什么限流 key 用 IP 不够？

因为多个用户可能在同一个 NAT 后面，共享一个出口 IP。只按 IP 限流会误伤。

生产中常见 key：

```text
用户 ID。
API Key。
租户 ID。
IP + User-Agent。
```

### 3. 固定窗口限流有什么问题？

窗口边界会有突刺。

例如第一秒最后 100ms 发 5 次，第二秒最开始 100ms 又发 5 次，短时间内实际来了 10 次。令牌桶和平滑滑动窗口能更好处理这个问题。

### 4. API 网关是不是一定要自己写？

通常不建议自己从零造生产网关。

实际项目常用 Nginx、Kong、APISIX、Envoy、Traefik、Spring Cloud Gateway 等。本节写 Go 版是为了理解网关内部做了什么。

---

## 十四、扩展练习

请继续完成：

1. 把 `routes` 改成读取 JSON 配置文件。
2. 为不同路由配置不同限流阈值。
3. 增加上游健康检查。
4. 增加 `X-Request-ID`，并转发给上游。
5. 区分 `401`、`403`、`429`、`502`、`504` 的返回场景。

---

## 十五、本节达标标准

学完本节后，你应该能够做到：

- 解释 API 网关和反向代理的区别。
- 用路径前缀匹配实现路由转发。
- 在网关层实现简单 Token 鉴权。
- 在网关层实现固定窗口限流。
- 说明限流 key 的选择问题。
- 使用 `ReverseProxy` 转发到不同业务服务。
- 用 curl 验证 `200`、`401`、`404`、`429`。
- 画出“请求进入网关到上游响应返回”的完整链路。

