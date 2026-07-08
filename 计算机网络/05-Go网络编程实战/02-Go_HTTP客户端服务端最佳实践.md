# 02-Go HTTP 客户端与服务端最佳实践

## 学习目标

掌握生产中常见的 Go HTTP Client 和 Server 配置，避免无超时、连接泄漏、连接池配置不当等问题。

## 一、不要随意使用默认 Client

下面代码能跑，但不适合作为生产服务中的通用写法：

```go
resp, err := http.Get("https://example.com")
```

问题：

- 超时控制不明确。
- 难以统一配置连接池。
- 难以加入日志、重试、指标。

## 二、推荐 HTTP Client

```go
var client = &http.Client{
    Timeout: 5 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 20,
        IdleConnTimeout:     90 * time.Second,
        TLSHandshakeTimeout: 3 * time.Second,
    },
}
```

关键点：

- `http.Client` 应复用，不要每次请求都新建。
- `Timeout` 控制整个请求生命周期。
- `Transport` 管理连接池。
- `MaxIdleConnsPerHost` 太小会限制并发复用能力。

## 三、正确关闭响应体

```go
resp, err := client.Do(req)
if err != nil {
    return err
}
defer resp.Body.Close()
```

如果不关闭 `resp.Body`：

- 连接无法回到连接池。
- 可能导致连接泄漏。
- 高并发下可能耗尽文件描述符。

如果希望连接复用，通常还要读完响应体：

```go
io.Copy(io.Discard, resp.Body)
resp.Body.Close()
```

## 四、使用 context 控制请求

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
if err != nil {
    return err
}

resp, err := client.Do(req)
```

`context` 的作用：

- 请求超时。
- 上游请求取消时，取消下游请求。
- 传递 trace id 等上下文信息。

## 五、推荐 HTTP Server

```go
srv := &http.Server{
    Addr:              ":8080",
    Handler:           router,
    ReadHeaderTimeout: 2 * time.Second,
    ReadTimeout:       5 * time.Second,
    WriteTimeout:      10 * time.Second,
    IdleTimeout:       60 * time.Second,
}
```

字段含义：

- `ReadHeaderTimeout`：读取请求头超时。
- `ReadTimeout`：读取整个请求超时。
- `WriteTimeout`：写响应超时。
- `IdleTimeout`：长连接空闲超时。

## 六、优雅关闭

服务收到退出信号时，应该停止接收新请求，并等待已有请求完成一段时间。

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

go func() {
    if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
        log.Fatal(err)
    }
}()

<-ctx.Done()

shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

if err := srv.Shutdown(shutdownCtx); err != nil {
    log.Println("shutdown error:", err)
}
```

## 七、常见错误清单

- 每次请求创建新的 `http.Client`。
- 不设置超时。
- 不关闭 `resp.Body`。
- 服务端不配置 `ReadHeaderTimeout`。
- 对所有错误都重试。
- 重试没有设置最大次数和退避。
- 没有记录上游请求耗时。

## 八、练习题

1. 为什么 `http.Client` 应该复用？
2. `resp.Body.Close()` 为什么重要？
3. `http.Client.Timeout` 和 `context.WithTimeout` 有什么关系？
4. Go HTTP Server 为什么需要优雅关闭？

## 九、验收标准

你能写出带超时、连接池、context、响应体关闭的 HTTP Client，也能写出带超时和优雅关闭的 HTTP Server。

