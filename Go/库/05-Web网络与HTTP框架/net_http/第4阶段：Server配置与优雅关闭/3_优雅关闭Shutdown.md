# 3. 优雅关闭 Shutdown

本节目标：理解为什么服务退出时不能总是直接杀进程，并学会使用 `Server.Shutdown` 做优雅关闭。

---

## 一、什么是优雅关闭

不优雅的关闭：

```text
进程立即退出。
正在处理的请求中断。
客户端可能收到连接断开。
日志和资源清理可能来不及完成。
```

优雅关闭：

```text
停止接收新请求。
等待正在处理的请求完成。
在限定时间内关闭空闲连接。
最后退出进程。
```

对于生产服务，发布、重启、扩缩容都会触发退出流程，所以优雅关闭很重要。

---

## 二、Shutdown 的基本用法

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

if err := srv.Shutdown(ctx); err != nil {
	log.Printf("server shutdown error: %v", err)
}
```

`Shutdown` 会：

- 关闭监听器，不再接收新连接。
- 关闭空闲连接。
- 等待活跃请求处理完成。
- 如果 context 超时，就返回错误。

---

## 三、为什么要给 Shutdown 设置超时

如果某个请求一直不结束，服务不能无限等待。

所以要给关闭过程设置上限：

```go
context.WithTimeout(context.Background(), 5*time.Second)
```

这表示最多等待 5 秒。

真实项目中可以根据接口特性设置，例如 5 秒、10 秒、30 秒。

---

## 四、ListenAndServe 和 Shutdown 如何配合

`ListenAndServe` 会阻塞当前 goroutine。如果要同时等待信号并关闭服务，需要把服务启动放到 goroutine 中：

```go
go func() {
	log.Println("server listening on :8080")
	if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
		log.Fatalf("server error: %v", err)
	}
}()
```

主 goroutine 等待退出信号，收到后调用 `Shutdown`。

---

## 五、模拟慢请求

加一个慢接口：

```go
mux.HandleFunc("GET /slow", func(w http.ResponseWriter, r *http.Request) {
	time.Sleep(3 * time.Second)
	fmt.Fprintln(w, "slow done")
})
```

启动服务后访问：

```bash
curl -i http://localhost:8080/slow
```

请求处理中按 `Ctrl+C`。如果优雅关闭等待时间足够，服务会等 `/slow` 返回后再退出。

---

## 六、Shutdown 和 Close 的区别

`Shutdown`：

- 优雅关闭。
- 等待正在处理的请求。
- 推荐用于正常退出。

`Close`：

- 立即关闭。
- 活跃连接也会被断开。
- 更像强制停止。

大多数业务服务应该优先使用 `Shutdown`。

---

## 七、本节检查点

请确认你能回答：

- 优雅关闭要解决什么问题？
- `Shutdown` 会停止接收新请求吗？
- 为什么 `Shutdown` 要传入带超时的 context？
- `Shutdown` 和 `Close` 有什么区别？

