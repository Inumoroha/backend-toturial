# 1. net/http 服务端与客户端 Context

本节目标：把服务端 `r.Context()` 和客户端 `NewRequestWithContext` 串起来。

在真实后端服务里，一个接口经常既是服务端，又是客户端：

```text
接收用户请求。
调用下游 HTTP 服务。
返回聚合结果。
```

这时 context 应该从入口传到下游请求。

---

## 一、服务端入口

```go
func profileHandler(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	_ = ctx
}
```

`r.Context()` 会随请求取消。

如果客户端断开连接，下游代码也应该尽快停止。

---

## 二、客户端请求

```go
func callUserService(ctx context.Context, userID string) error {
	ctx, cancel := context.WithTimeout(ctx, 500*time.Millisecond)
	defer cancel()

	req, err := http.NewRequestWithContext(ctx, http.MethodGet, "http://localhost:8081/users/"+userID, nil)
	if err != nil {
		return err
	}

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return fmt.Errorf("user service status: %s", resp.Status)
	}

	return nil
}
```

这个函数接收上游 ctx，并为当前下游调用派生 500ms timeout。

---

## 三、Handler 串联下游调用

```go
func profileHandler(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), time.Second)
	defer cancel()

	if err := callUserService(ctx, "1001"); err != nil {
		http.Error(w, err.Error(), http.StatusBadGateway)
		return
	}

	fmt.Fprintln(w, "ok")
}
```

这里有两层时间边界：

```text
整个 profile 接口最多 1 秒。
用户服务调用最多 500ms。
```

---

## 四、服务端超时配置

启动 HTTP server 时建议配置超时：

```go
server := &http.Server{
	Addr:              ":8080",
	Handler:           mux,
	ReadHeaderTimeout: 3 * time.Second,
	ReadTimeout:       5 * time.Second,
	WriteTimeout:      10 * time.Second,
	IdleTimeout:       60 * time.Second,
}
```

这些是网络层和连接层保护。

业务链路仍然要传 context。

---

## 五、本节达标标准

学完本节后，你应该能够做到：

- 在 handler 中获取请求 context。
- 把请求 context 传给 HTTP 客户端调用。
- 为下游 HTTP 调用派生更短 timeout。
- 知道 HTTP server 超时配置和 context 不是一回事。

---

## 六、完整可运行示例

下面这个示例同时启动两个服务：

```text
:8081 模拟 user-service。
:8080 提供 profile 接口，并调用 user-service。
```

```go
package main

import (
	"context"
	"fmt"
	"net/http"
	"time"
)

func userService() {
	mux := http.NewServeMux()
	mux.HandleFunc("/users/1001", func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(300 * time.Millisecond)
		fmt.Fprintln(w, `{"id":1001,"name":"Tom"}`)
	})

	go func() {
		fmt.Println("user service listen on :8081")
		_ = http.ListenAndServe(":8081", mux)
	}()
}

func callUserService(ctx context.Context) error {
	ctx, cancel := context.WithTimeout(ctx, 500*time.Millisecond)
	defer cancel()

	req, err := http.NewRequestWithContext(ctx, http.MethodGet, "http://localhost:8081/users/1001", nil)
	if err != nil {
		return err
	}

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return fmt.Errorf("status: %s", resp.Status)
	}

	return nil
}

func profileHandler(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), time.Second)
	defer cancel()

	if err := callUserService(ctx); err != nil {
		http.Error(w, err.Error(), http.StatusBadGateway)
		return
	}

	fmt.Fprintln(w, "profile ok")
}

func main() {
	userService()

	mux := http.NewServeMux()
	mux.HandleFunc("/profile/1001", profileHandler)

	server := &http.Server{
		Addr:              ":8080",
		Handler:           mux,
		ReadHeaderTimeout: 3 * time.Second,
	}

	fmt.Println("api service listen on :8080")
	_ = server.ListenAndServe()
}
```

运行：

```powershell
go run .
```

访问：

```powershell
curl http://localhost:8080/profile/1001
```

---

## 七、观察超时

把 user-service 的耗时改成：

```go
time.Sleep(800 * time.Millisecond)
```

因为 `callUserService` 只给了 500ms，所以 profile 会失败。

你可以观察错误信息是否和 context deadline 相关。

---

## 八、常见错误

### 1. 使用 http.Get

```go
resp, err := http.Get(url)
```

这样没有传入请求级 context。

更推荐：

```go
req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
resp, err := client.Do(req)
```

### 2. 下游函数内部使用 Background

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
```

这会丢掉客户端断开连接的信号。

### 3. 忘记关闭响应体

```go
defer resp.Body.Close()
```

HTTP 响应体不关闭，会影响连接复用。

---

## 九、本节练习

请扩展示例：

1. 增加一个 `/slow-users/1001`，耗时 2 秒。
2. profile 调用它时设置 700ms timeout。
3. 在日志中打印 `r.Context().Err()`。
4. 使用浏览器访问后中途取消请求。
5. 观察服务端是否能感知取消。
