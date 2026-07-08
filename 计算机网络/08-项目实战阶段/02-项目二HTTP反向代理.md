# 02-项目二：HTTP 反向代理

## 项目目标

实现一个简易 HTTP 反向代理。这个项目用于理解 HTTP 转发、Header 处理、超时、连接池、错误码和请求日志。

## 一、功能要求

基础功能：

- 接收客户端 HTTP 请求。
- 转发到一个后端服务。
- 返回后端响应。
- 记录请求方法、路径、状态码、耗时。
- 支持超时控制。

进阶功能：

- 支持多个后端。
- 轮询负载均衡。
- 健康检查。
- X-Request-Id。
- 错误时返回 502 或 504。

## 二、整体流程

```text
客户端
  -> 代理服务
    -> 后端服务
    <- 后端响应
  <- 代理返回响应
```

## 三、使用标准库 ReverseProxy

Go 标准库提供：

```go
net/http/httputil.ReverseProxy
```

基础示例：

```go
target, _ := url.Parse("http://127.0.0.1:8081")

proxy := httputil.NewSingleHostReverseProxy(target)

server := &http.Server{
    Addr:              ":8080",
    Handler:           proxy,
    ReadHeaderTimeout: 2 * time.Second,
    ReadTimeout:       5 * time.Second,
    WriteTimeout:      10 * time.Second,
    IdleTimeout:       60 * time.Second,
}

server.ListenAndServe()
```

## 四、自定义 Transport

```go
transport := &http.Transport{
    MaxIdleConns:          100,
    MaxIdleConnsPerHost:   20,
    MaxConnsPerHost:       100,
    IdleConnTimeout:       90 * time.Second,
    TLSHandshakeTimeout:   3 * time.Second,
    ResponseHeaderTimeout: 5 * time.Second,
}

proxy.Transport = transport
```

## 五、错误处理

```go
proxy.ErrorHandler = func(w http.ResponseWriter, r *http.Request, err error) {
    http.Error(w, "bad gateway", http.StatusBadGateway)
}
```

如果是超时，可以返回 504：

```go
if errors.Is(err, context.DeadlineExceeded) {
    http.Error(w, "gateway timeout", http.StatusGatewayTimeout)
    return
}
```

## 六、请求日志中间件

```go
func logging(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s cost=%s", r.Method, r.URL.Path, time.Since(start))
    })
}
```

实际项目中还应该记录状态码，可以封装 `ResponseWriter`。

## 七、轮询负载均衡思路

```go
type Balancer struct {
    mu      sync.Mutex
    targets []*url.URL
    next    int
}

func (b *Balancer) Pick() *url.URL {
    b.mu.Lock()
    defer b.mu.Unlock()

    target := b.targets[b.next%len(b.targets)]
    b.next++
    return target
}
```

每次请求选择一个后端，然后修改请求目标地址。

## 八、测试方式

启动两个后端：

```bash
PORT=8081 APP_NAME=backend-1 go run .
PORT=8082 APP_NAME=backend-2 go run .
```

启动代理：

```bash
go run ./cmd/proxy
```

请求：

```bash
curl -v http://127.0.0.1:8080
```

## 九、验收标准

必须完成：

- 能代理请求到后端。
- 后端关闭时代理返回合理错误。
- 请求日志包含方法、路径、耗时。
- 代理和上游调用都有超时。

加分项：

- 多后端轮询。
- 健康检查。
- 支持请求 ID。
- 统计每个后端成功和失败次数。

