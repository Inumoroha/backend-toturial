# 5. 中间件与 Server 启动

本节目标：给 Todo API 组装中间件链，并使用生产级基础启动方式。

---

## 一、中间件链

```go
handler := middleware.Chain(
	routes,
	middleware.Recovery(logger),
	middleware.RequestID(),
	middleware.Logging(logger),
	middleware.CORS(allowedOrigins),
)
```

写操作可以单独加 Auth：

```go
mux.Handle("POST /todos", middleware.Auth(http.HandlerFunc(h.createTodo)))
```

---

## 二、Server 配置

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

## 三、优雅关闭

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

shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

if err := srv.Shutdown(shutdownCtx); err != nil {
	logger.Error("shutdown error", "error", err)
}
```

---

## 四、本节检查点

请确认你能做到：

- 把路由和中间件组合成最终 handler。
- 显式配置 `http.Server`。
- 捕获退出信号。
- 使用 `Shutdown` 优雅关闭。

