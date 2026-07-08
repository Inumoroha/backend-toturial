# 1. net/http 中的 Request Context

本节目标：理解 `r.Context()` 的来源和行为。

Go 标准库 `net/http` 中，每个请求都有自己的 context：

```go
ctx := r.Context()
```

这个 context 和当前 HTTP 请求生命周期绑定。

---

## 一、基本使用

```go
func handler(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	select {
	case <-time.After(2 * time.Second):
		fmt.Fprintln(w, "done")
	case <-ctx.Done():
		fmt.Println("request canceled:", ctx.Err())
	}
}
```

如果客户端断开连接，`r.Context()` 会被取消。

---

## 二、服务端超时也会影响请求

HTTP server 可以配置超时：

```go
server := &http.Server{
	Addr:         ":8080",
	Handler:      mux,
	ReadTimeout:  5 * time.Second,
	WriteTimeout: 10 * time.Second,
	IdleTimeout:  60 * time.Second,
}
```

这些超时和请求 context 一起构成服务端的生命周期控制。

context 更适合在业务链路中向下游传播取消信号。

---

## 三、不要把 r 传到所有层

不推荐：

```go
func (s *UserService) GetUser(r *http.Request, id int64) (*User, error) {
	ctx := r.Context()
	return s.repo.FindByID(ctx, id)
}
```

service 不应该依赖 HTTP 类型。

推荐：

```go
func (s *UserService) GetUser(ctx context.Context, id int64) (*User, error) {
	return s.repo.FindByID(ctx, id)
}
```

handler 负责从 HTTP 请求中取出 ctx，然后传给 service。

这样 service 也能被 CLI、测试、消息队列消费者复用。

---

## 四、给单个请求派生业务超时

```go
func handler(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), 800*time.Millisecond)
	defer cancel()

	if err := doBusiness(ctx); err != nil {
		http.Error(w, err.Error(), http.StatusGatewayTimeout)
		return
	}

	fmt.Fprintln(w, "ok")
}
```

注意是基于 `r.Context()` 派生，而不是 `context.Background()`。

---

## 五、本节练习

请写一个 HTTP 服务：

1. `GET /slow` 模拟 3 秒任务。
2. handler 中监听 `r.Context().Done()`。
3. 用浏览器访问后中途关闭页面。
4. 观察服务端日志是否打印请求取消。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 在 handler 中使用 `r.Context()`。
- 理解客户端断开连接会取消 request context。
- 不把 `*http.Request` 传进 service。
- 基于 `r.Context()` 派生请求级超时。

---

## 七、完整可运行示例

```go
package main

import (
	"fmt"
	"net/http"
	"time"
)

func slowHandler(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	fmt.Println("request start")

	select {
	case <-time.After(5 * time.Second):
		fmt.Fprintln(w, "slow done")
	case <-ctx.Done():
		fmt.Println("request stopped:", ctx.Err())
	}
}

func main() {
	http.HandleFunc("/slow", slowHandler)
	fmt.Println("listen on :8080")
	_ = http.ListenAndServe(":8080", nil)
}
```

运行：

```powershell
go run .
```

访问：

```powershell
curl http://localhost:8080/slow
```

请求没完成前按 Ctrl+C 停止 curl，观察服务端日志。

---

## 八、给 handler 加业务 timeout

```go
func slowHandler(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), time.Second)
	defer cancel()

	select {
	case <-time.After(5 * time.Second):
		fmt.Fprintln(w, "done")
	case <-ctx.Done():
		http.Error(w, ctx.Err().Error(), http.StatusGatewayTimeout)
	}
}
```

这个 timeout 是业务层的。

客户端不断开，1 秒后也会超时。

---

## 九、常见错误

### 1. handler 中使用 Background

```go
ctx := context.Background()
```

这会忽略请求生命周期。

### 2. 把 request 传给 service

service 不应该知道 HTTP。

### 3. 忘记处理 ctx.Done

只拿到 ctx 不使用，取消不会产生实际效果。

---

## 十、本节练习

请写两个接口：

```text
GET /fast：100ms 返回。
GET /slow：3s 返回，但 handler timeout 是 1s。
```

要求：

1. 都使用 `r.Context()`。
2. `/slow` 超时时返回 504。
3. 客户端取消时服务端打印日志。
