# 10. 组装 main 与优雅关闭

本节目标：把 Store、Service、Handler、Middleware 和 `http.Server` 组装成可运行服务。

---

## 一、main.go 的职责

`cmd/server/main.go` 不应该堆业务逻辑。它主要负责：

```text
创建 logger
创建 store
创建 service
创建 handler
注册 middleware
配置 http.Server
启动服务
处理优雅关闭
```

---

## 二、组装依赖

```go
logger := slog.New(slog.NewTextHandler(os.Stdout, nil))

store := todo.NewMemoryStore()
service := todo.NewService(store)
api := httpapi.NewHandler(service)
routes := api.Routes()
```

导入路径要和你的 module 名称一致。

---

## 三、组合中间件

```go
handler := middleware.Chain(
	routes,
	middleware.Recovery(logger),
	middleware.RequestID(),
	middleware.Logging(logger),
)
```

如果你给写操作单独加 Auth，可以在 `Routes()` 内部对具体路由包装。

---

## 四、配置 Server

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

---

## 五、启动和关闭

```go
go func() {
	logger.Info("server started", "addr", srv.Addr)
	if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
		logger.Error("server error", "error", err)
		os.Exit(1)
	}
}()

ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

<-ctx.Done()
logger.Info("shutdown signal received")

shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

if err := srv.Shutdown(shutdownCtx); err != nil {
	logger.Error("shutdown error", "error", err)
}

logger.Info("server stopped")
```

---

## 六、验证启动

运行：

```bash
go run ./cmd/server
```

访问：

```bash
curl -i http://localhost:8080/health
```

按 `Ctrl+C`，观察日志中是否出现：

```text
shutdown signal received
server stopped
```

---

## 七、本节检查点

你应该能解释：

- `main.go` 为什么只做组装。
- 为什么 `ListenAndServe` 要放 goroutine。
- 为什么要忽略 `http.ErrServerClosed`。
- `Shutdown` 最多等待多久。
