# 9. 组装 main 与中间件

本节目标：把短链接项目组装成可运行服务，并复用前面学过的 Server 和 Middleware 写法。

---

## 一、初始化项目

```bash
mkdir shortener
cd shortener
go mod init example.com/shortener
```

目录：

```text
shortener/
  cmd/server/main.go
  internal/link/
  internal/httpapi/
  internal/middleware/
```

---

## 二、main.go 组装依赖

```go
logger := slog.New(slog.NewTextHandler(os.Stdout, nil))

store := link.NewMemoryStore()
service := link.NewService(store)
api := httpapi.NewHandler(service, "http://localhost:8080")
routes := api.Routes()
```

---

## 三、复用中间件

短链接服务至少需要：

- Recovery。
- Request ID。
- Logging。
- CORS，可选。

```go
handler := middleware.Chain(
	routes,
	middleware.Recovery(logger),
	middleware.RequestID(),
	middleware.Logging(logger),
)
```

创建短链接接口如果要保护，可以给 `POST /links` 加 Auth。跳转接口通常公开。

---

## 四、Server 配置

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

## 五、优雅关闭

短链接跳转接口可能很多，发布重启时更应该优雅关闭：

```go
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

## 六、本节检查点

你应该能解释：

- 为什么跳转接口通常公开。
- 为什么创建接口可以加鉴权。
- 短链接服务为什么也需要超时配置。
- `baseURL` 为什么应该来自配置。
