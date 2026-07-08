# 3. HTTP 客户端请求超时

本节目标：掌握 `http.NewRequestWithContext`，让 HTTP 下游调用受 context 控制。

后端服务经常调用其他 HTTP 服务。

如果下游服务慢、网络卡住或连接迟迟不返回，请求可能一直等待。

正确做法是给请求传入 context。

---

## 一、基础写法

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()

req, err := http.NewRequestWithContext(ctx, http.MethodGet, "https://example.com", nil)
if err != nil {
	return err
}

resp, err := http.DefaultClient.Do(req)
if err != nil {
	return err
}
defer resp.Body.Close()
```

当 context 超时或取消时，HTTP 请求也会被取消。

---

## 二、不要丢掉上游 context

在 HTTP handler 中调用下游服务时，不要这样：

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
```

这会切断上游请求的取消信号。

应该基于 `r.Context()` 派生：

```go
ctx, cancel := context.WithTimeout(r.Context(), time.Second)
defer cancel()
```

这样如果客户端断开连接，下游 HTTP 请求也能被取消。

---

## 三、封装 HTTP 客户端函数

```go
func fetchUser(ctx context.Context, client *http.Client, userID string) error {
	ctx, cancel := context.WithTimeout(ctx, 800*time.Millisecond)
	defer cancel()

	url := "https://api.example.com/users/" + userID
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return err
	}

	resp, err := client.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	if resp.StatusCode >= 500 {
		return fmt.Errorf("upstream server error: %s", resp.Status)
	}

	return nil
}
```

注意函数接收上游传来的 `ctx`，然后为这个具体下游调用派生更短超时。

---

## 四、http.Client 自身也有 Timeout

`http.Client` 也可以设置：

```go
client := &http.Client{
	Timeout: 2 * time.Second,
}
```

它是客户端级别的总超时。

实际项目中可以同时使用：

- `http.Client.Timeout`：兜底保护。
- `request context`：每次请求的链路级控制。

不要只依赖一个全局很长的 client timeout，而忽略每次请求自己的 context。

---

## 五、错误处理

HTTP 请求因为 context 超时失败时，可以判断：

```go
if errors.Is(err, context.DeadlineExceeded) {
	return fmt.Errorf("fetch user timeout: %w", err)
}
```

但有些网络错误会被包装在更复杂的错误里。

实际项目中通常还会结合日志和状态码处理。

核心原则是：

```text
下游超时要和普通业务失败区分开。
```

---

## 六、本节练习

请完成：

1. 写一个 `fetch(ctx context.Context, url string) error` 函数。
2. 使用 `http.NewRequestWithContext`。
3. 给请求设置 1 秒 timeout。
4. 请求一个你能访问的 URL。
5. 把 timeout 改成极短，例如 1ms，观察错误。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 使用 `http.NewRequestWithContext`。
- 在 handler 中基于 `r.Context()` 派生下游超时。
- 知道 `http.Client.Timeout` 和 request context 的区别。
- 下游 HTTP 请求失败时能考虑是否是 context 超时。

---

## 八、完整可运行示例

创建一个慢服务和调用方：

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"net/http"
	"time"
)

func startSlowServer() {
	mux := http.NewServeMux()
	mux.HandleFunc("/slow", func(w http.ResponseWriter, r *http.Request) {
		select {
		case <-time.After(2 * time.Second):
			fmt.Fprintln(w, "slow response")
		case <-r.Context().Done():
			fmt.Println("slow server request canceled:", r.Context().Err())
		}
	})

	go http.ListenAndServe(":8081", mux)
}

func fetch(ctx context.Context, url string) error {
	ctx, cancel := context.WithTimeout(ctx, 500*time.Millisecond)
	defer cancel()

	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return err
	}

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	return nil
}

func main() {
	startSlowServer()
	time.Sleep(100 * time.Millisecond)

	err := fetch(context.Background(), "http://localhost:8081/slow")
	if errors.Is(err, context.DeadlineExceeded) {
		fmt.Println("client timeout")
		return
	}
	if err != nil {
		fmt.Println("client error:", err)
		return
	}
}
```

运行后，客户端会超时，慢服务也可能感知请求取消。

---

## 九、Client Timeout 和 Context Timeout 怎么配合

建议：

```go
client := &http.Client{
	Timeout: 5 * time.Second,
}
```

作为全局兜底。

每次请求：

```go
ctx, cancel := context.WithTimeout(ctx, 800*time.Millisecond)
defer cancel()
```

作为业务预算。

如果业务预算更短，它会先触发。

---

## 十、常见错误

### 1. 只设置 client.Timeout

这样不能表达每个请求自己的预算，也不能很好地跟上游请求取消联动。

### 2. 使用 context.Background

在 handler 中调用下游时，必须基于 `r.Context()`。

### 3. 不关闭 resp.Body

即使不读取 body，也要关闭。

### 4. 把所有错误都当 timeout

网络错误、DNS 错误、连接拒绝都可能导致 `client.Do` 返回错误，不一定是 context 超时。

---

## 十一、本节练习

请写一个下游 HTTP client：

```go
type UserClient struct {
	client *http.Client
	baseURL string
}
```

实现：

```go
func (c *UserClient) GetUser(ctx context.Context, id int64) error
```

要求：

1. 基于上游 ctx 派生 500ms timeout。
2. 使用 `NewRequestWithContext`。
3. 检查 HTTP status。
4. 关闭 response body。
5. 超时时保留 `DeadlineExceeded` 根因。
