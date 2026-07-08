# 5. 阶段练习：改造 Todo 服务启动方式

本节目标：把前面写的 Todo API 从简写启动方式，改造成带超时和优雅关闭的生产级基础启动方式。

---

## 一、改造目标

原来可能是：

```go
log.Fatal(http.ListenAndServe(":8080", handler))
```

现在改成：

- 显式创建 `http.Server`。
- 配置超时。
- goroutine 中启动服务。
- 主 goroutine 捕获退出信号。
- 收到信号后调用 `Shutdown`。
- 正确处理 `http.ErrServerClosed`。

---

## 二、参考启动函数

可以把启动逻辑放在 `main` 中：

```go
srv := &http.Server{
	Addr:              ":8080",
	Handler:           handler,
	ReadHeaderTimeout: 5 * time.Second,
	ReadTimeout:       10 * time.Second,
	WriteTimeout:      10 * time.Second,
	IdleTimeout:       60 * time.Second,
	MaxHeaderBytes:    1 << 20,
}

go func() {
	log.Println("server listening on :8080")
	if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
		log.Fatalf("server error: %v", err)
	}
}()

ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

<-ctx.Done()
log.Println("shutdown signal received")

shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

if err := srv.Shutdown(shutdownCtx); err != nil {
	log.Printf("server shutdown error: %v", err)
}

log.Println("server stopped")
```

---

## 三、添加慢接口验证优雅关闭

临时添加：

```go
mux.HandleFunc("GET /slow", func(w http.ResponseWriter, r *http.Request) {
	time.Sleep(3 * time.Second)
	fmt.Fprintln(w, "slow done")
})
```

验证步骤：

1. 启动服务。
2. 访问 `/slow`。
3. 请求未结束时按 `Ctrl+C`。
4. 观察服务是否等待请求完成。

命令：

```bash
curl -i http://localhost:8080/slow
```

---

## 四、测试超时配置

你可以先观察配置是否影响正常请求：

```bash
curl -i http://localhost:8080/health
curl -i http://localhost:8080/todos
```

如果普通请求正常，说明基础配置没有破坏服务。

慢连接测试可以后续用专门工具进行。当前阶段重点是理解配置含义和掌握写法。

---

## 五、阶段复盘

完成本阶段后，请写下：

- 你的服务有哪些超时配置？
- 每个超时配置保护哪个阶段？
- `Shutdown` 最多等待多久？
- 如果端口被占用，日志会是什么？
- 为什么 `http.ErrServerClosed` 不应该被当成 fatal 错误？

完成这些后，你的服务已经从“学习示例”向“生产基础”迈出关键一步。下一阶段进入 `context.Context`，理解请求取消和超时如何贯穿业务代码。

