# 3. signal.NotifyContext 与优雅关闭

本节目标：使用 `signal.NotifyContext` 管理服务退出信号，并理解优雅关闭流程。

后端服务不能只靠强制退出。

收到退出信号后，应该：

```text
停止接收新请求。
等待正在处理的请求结束。
通知后台 worker 停止。
关闭数据库连接池。
最后退出进程。
```

---

## 一、基础用法

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

<-ctx.Done()
fmt.Println("shutdown signal received")
```

当进程收到 Ctrl+C 或 SIGTERM 时，ctx 会被取消。

---

## 二、HTTP Server 优雅关闭

```go
server := &http.Server{
	Addr:    ":8080",
	Handler: mux,
}

go func() {
	if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
		log.Fatal(err)
	}
}()

ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

<-ctx.Done()

shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

if err := server.Shutdown(shutdownCtx); err != nil {
	log.Println("shutdown error:", err)
}
```

`Shutdown` 会停止接收新连接，并等待已有请求完成。

---

## 三、为什么 Shutdown 用新的 context

收到退出信号后，`ctx` 已经取消。

如果直接把它传给 `server.Shutdown(ctx)`，可能会立刻失败。

所以通常创建一个新的 shutdown context：

```go
shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
```

它表示：

```text
最多等待 5 秒完成优雅关闭。
```

---

## 四、通知 worker 停止

可以把 signal context 传给 worker：

```go
func worker(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			return
		default:
			doWork()
		}
	}
}
```

主程序收到信号后，worker 会退出。

---

## 五、本节达标标准

学完本节后，你应该能够做到：

- 使用 `signal.NotifyContext`。
- 收到退出信号后调用 `server.Shutdown`。
- 为 Shutdown 单独设置超时。
- 通知后台 worker 停止。

---

## 六、完整可运行 HTTP 示例

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/slow", func(w http.ResponseWriter, r *http.Request) {
		select {
		case <-time.After(3 * time.Second):
			fmt.Fprintln(w, "done")
		case <-r.Context().Done():
			log.Println("request canceled:", r.Context().Err())
		}
	})

	server := &http.Server{
		Addr:              ":8080",
		Handler:           mux,
		ReadHeaderTimeout: 3 * time.Second,
	}

	go func() {
		log.Println("listen on :8080")
		if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatal(err)
		}
	}()

	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	<-ctx.Done()
	log.Println("shutdown signal received")

	shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	if err := server.Shutdown(shutdownCtx); err != nil {
		log.Println("server shutdown error:", err)
	}

	log.Println("server stopped")
}
```

运行：

```powershell
go run .
```

按 Ctrl+C，观察服务是否优雅退出。

---

## 七、Shutdown 和 Close 的区别

`server.Shutdown(ctx)`：

```text
停止接收新请求。
等待正在处理的请求完成。
ctx 超时后结束等待。
```

`server.Close()`：

```text
立即关闭监听器和活动连接。
更粗暴。
```

真实服务优先使用 `Shutdown`。

---

## 八、worker 和 HTTP 一起关闭

很多服务既有 HTTP server，也有后台 worker。

根 ctx 可以传给 worker：

```go
go worker(ctx)
```

收到信号后：

```text
HTTP server 停止接收新请求。
worker 收到 ctx.Done 后退出。
主进程等待一小段时间。
```

如果 worker 需要处理完当前任务，可以结合 WaitGroup。

---

## 九、常见错误

### 1. 直接 os.Exit

直接退出会跳过 defer，资源可能来不及释放。

### 2. Shutdown 使用 signal ctx

signal ctx 已经取消，不能作为优雅关闭等待上下文。

### 3. 没有设置 Shutdown timeout

如果某个请求一直不结束，关闭流程可能一直卡住。

### 4. 只关闭 HTTP，不管 worker

后台任务可能仍然运行，导致进程无法退出或数据不一致。

---

## 十、本节练习

请扩展示例：

1. 增加一个 worker，每秒打印一次。
2. worker 监听 signal ctx。
3. Ctrl+C 后 HTTP server 和 worker 都退出。
4. 使用 WaitGroup 等待 worker 退出。
5. 给整体关闭流程设置 5 秒 timeout。
