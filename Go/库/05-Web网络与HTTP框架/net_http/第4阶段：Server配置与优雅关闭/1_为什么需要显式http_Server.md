# 1. 为什么需要显式 http.Server

本节目标：理解为什么真实项目中更推荐显式创建 `http.Server`，而不是一直使用 `http.ListenAndServe` 的简写形式。

---

## 一、简写形式做了什么

前面我们一直写：

```go
log.Fatal(http.ListenAndServe(":8080", mux))
```

它很方便，本质上会创建一个默认配置的 Server 并启动。

学习阶段没有问题，但它隐藏了很多生产环境关心的配置，例如：

- 读 Header 超时。
- 读请求体超时。
- 写响应超时。
- 空闲连接超时。
- 最大 Header 大小。
- 关闭流程。

如果这些都不配置，服务在异常客户端、慢请求、大请求或发布重启时会更脆弱。

---

## 二、显式 Server 写法

推荐基础写法：

```go
srv := &http.Server{
	Addr:              ":8080",
	Handler:           mux,
	ReadHeaderTimeout: 5 * time.Second,
	ReadTimeout:       10 * time.Second,
	WriteTimeout:      10 * time.Second,
	IdleTimeout:       60 * time.Second,
	MaxHeaderBytes:    1 << 20,
}

log.Println("server listening on :8080")
if err := srv.ListenAndServe(); err != nil {
	log.Fatal(err)
}
```

需要导入：

```go
import "time"
```

---

## 三、字段解释

`Addr`：

```go
Addr: ":8080"
```

监听地址。`":8080"` 表示监听本机所有网卡的 8080 端口。

`Handler`：

```go
Handler: mux
```

请求进入后交给谁处理。通常是 mux 或被中间件包装后的 handler。

`ReadHeaderTimeout`：

```go
ReadHeaderTimeout: 5 * time.Second
```

限制读取请求头的时间。防止客户端慢慢发送 Header 占住连接。

`ReadTimeout`：

```go
ReadTimeout: 10 * time.Second
```

限制读取整个请求的时间，包括 Header 和 Body。

`WriteTimeout`：

```go
WriteTimeout: 10 * time.Second
```

限制写响应的时间。防止服务端一直卡在写响应。

`IdleTimeout`：

```go
IdleTimeout: 60 * time.Second
```

限制 keep-alive 空闲连接保留多久。

`MaxHeaderBytes`：

```go
MaxHeaderBytes: 1 << 20
```

限制请求头最大大小。`1 << 20` 是 1MB。

---

## 四、为什么超时是安全能力

HTTP 服务面对的是网络。客户端可能：

- 网络很慢。
- 故意慢慢发送数据。
- 上传特别大的请求。
- 连接后不发完整请求。
- 接收响应很慢。

如果服务端没有超时，连接和 goroutine 就可能长期被占住。

所以超时不是性能优化的小细节，而是服务稳定性的基础。

---

## 五、ListenAndServe 返回什么错误

启动失败时，比如端口被占用：

```text
listen tcp :8080: bind: address already in use
```

`ListenAndServe` 会返回错误。

正常调用 `Shutdown` 关闭服务时，它会返回：

```go
http.ErrServerClosed
```

所以优雅关闭版本通常会这样判断：

```go
if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
	log.Fatalf("server error: %v", err)
}
```

不要把 `http.ErrServerClosed` 当成异常崩溃。

---

## 六、本节检查点

请确认你能回答：

- `http.ListenAndServe` 简写形式适合什么场景？
- 显式 `http.Server` 能配置哪些重要参数？
- 为什么 `ReadHeaderTimeout` 很重要？
- `http.ErrServerClosed` 表示什么？

