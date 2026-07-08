# 4. HTTP Server 超时配置与中间件

本节目标：写一个具备基础生产意识的 Go HTTP Server，包括超时配置、日志中间件和健康检查。

---

## 一、推荐 Server 配置

```go
srv := &http.Server{
    Addr:              ":8080",
    Handler:           handler,
    ReadHeaderTimeout: 2 * time.Second,
    ReadTimeout:       5 * time.Second,
    WriteTimeout:      10 * time.Second,
    IdleTimeout:       60 * time.Second,
}
```

不要只写：

```go
http.ListenAndServe(":8080", nil)
```

因为默认超时不清晰。

---

## 二、完整示例

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
    mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusNoContent)
    })
    mux.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "hello")
    })

    srv := &http.Server{
        Addr:              ":8080",
        Handler:           logging(mux),
        ReadHeaderTimeout: 2 * time.Second,
        ReadTimeout:       5 * time.Second,
        WriteTimeout:      10 * time.Second,
        IdleTimeout:       60 * time.Second,
    }

    log.Fatal(srv.ListenAndServe())
}

func logging(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("method=%s path=%s cost=%s", r.Method, r.URL.Path, time.Since(start))
    })
}
```

---

## 三、测试

```bash
curl -i http://127.0.0.1:8080/health
curl -v http://127.0.0.1:8080/hello
```

查看监听：

```bash
ss -lntp | grep 8080
```

---

## 补充实验：慢请求头为什么危险

`ReadHeaderTimeout` 是为了防止客户端很慢地发送请求头。

你可以用 `nc` 模拟：

```bash
nc 127.0.0.1 8080
```

只输入一部分：

```http
GET / HTTP/1.1
Host: localhost
```

然后故意不输入空行。没有 `ReadHeaderTimeout` 时，服务端可能长时间保留这个连接。

生产中如果大量慢连接占住 fd 和 goroutine，正常请求也会受影响。

---

## 补充实践：中间件应该记录什么

基础日志中间件至少记录：

```text
method
path
status
duration
remote_addr
request_id
```

如果没有状态码记录，排障时很难统计：

```text
到底是 2xx 慢？
还是 5xx 多？
还是 499/客户端断开多？
```

所以 HTTP Server 教程不只是会启动服务，还要从一开始养成可观测性习惯。

---

## 补充检查：Server 配置不要散落

项目里建议统一创建 `http.Server`：

```go
srv := &http.Server{
    Addr:              ":8080",
    Handler:           handler,
    ReadHeaderTimeout: 3 * time.Second,
    ReadTimeout:       10 * time.Second,
    WriteTimeout:      10 * time.Second,
    IdleTimeout:       60 * time.Second,
}
```

不要在不同文件里随手写多个 `http.ListenAndServe`。统一入口更容易做：

```text
超时配置。
优雅关闭。
日志中间件。
panic recover。
metrics。
```

这就是工程化 HTTP Server 和临时 demo 的分界线。

---

## 四、常见问题

### 1. ReadHeaderTimeout 有什么用？

防止客户端非常慢地发送 Header，占用连接。

### 2. WriteTimeout 会影响大文件下载吗？

会。流式或大文件场景要谨慎设置和测试。

### 3. 中间件本质是什么？

接收一个 `http.Handler`，返回一个新的 `http.Handler`。

---

## 五、本节达标标准

学完本节后，你应该能够做到：

- 写出带超时配置的 HTTP Server。
- 实现简单日志中间件。
- 提供 `/health` 健康检查。
- 使用 curl 和 ss 验证服务。
