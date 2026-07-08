# 0. Server 配置与优雅关闭总览

本阶段目标：从 `http.ListenAndServe(":8080", handler)` 过渡到显式配置 `http.Server`。生产环境的 HTTP 服务不能只会启动，还要能设置超时、限制资源占用，并在收到退出信号时优雅关闭。

---

## 一、本阶段要解决什么问题

第 1 阶段使用过：

```go
http.ListenAndServe(":8080", mux)
```

它适合学习和快速演示，但真实项目中通常需要：

- 配置读取 Header 的超时。
- 配置读取 Body 的超时。
- 配置写响应的超时。
- 配置空闲连接超时。
- 处理启动错误。
- 收到 `Ctrl+C` 或容器停止信号后优雅退出。

这些都需要显式创建 `http.Server`。

---

## 二、本阶段文档安排

建议按下面顺序学习：

1. `1_为什么需要显式http_Server.md`
2. `2_超时配置详解.md`
3. `3_优雅关闭Shutdown.md`
4. `4_信号处理与服务生命周期.md`
5. `5_阶段练习_改造Todo服务启动方式.md`

---

## 三、本阶段核心代码

```go
srv := &http.Server{
	Addr:              ":8080",
	Handler:           handler,
	ReadHeaderTimeout: 5 * time.Second,
	ReadTimeout:       10 * time.Second,
	WriteTimeout:      10 * time.Second,
	IdleTimeout:       60 * time.Second,
}
```

优雅关闭：

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

if err := srv.Shutdown(ctx); err != nil {
	log.Printf("server shutdown error: %v", err)
}
```

---

## 四、本阶段达标标准

完成本阶段后，你应该能：

- 解释为什么生产服务要配置超时。
- 使用 `http.Server` 启动服务。
- 正确区分 `http.ErrServerClosed`。
- 使用 `os/signal` 捕获退出信号。
- 使用 `Shutdown` 等待正在处理的请求完成。

