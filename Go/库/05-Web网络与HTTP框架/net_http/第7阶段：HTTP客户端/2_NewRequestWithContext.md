# 2. NewRequestWithContext

本节目标：学会为客户端请求绑定 context，让调用能响应取消和超时。

---

## 一、为什么客户端请求要带 context

后端服务调用第三方 API 时，不能只写：

```go
http.Get(url)
```

因为你需要控制：

- 当前请求取消后，下游调用也取消。
- 当前接口只允许等待下游一小段时间。
- 服务关闭时，正在进行的调用能尽快结束。

---

## 二、基本写法

```go
req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
if err != nil {
	return err
}

resp, err := client.Do(req)
if err != nil {
	return err
}
defer resp.Body.Close()
```

如果在 Handler 中调用：

```go
func handler(w http.ResponseWriter, r *http.Request) {
	resp, err := apiClient.GetUser(r.Context(), "alice")
}
```

Client 方法：

```go
func (c *APIClient) GetUser(ctx context.Context, username string) (*User, error) {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, c.baseURL+"/users/"+username, nil)
	if err != nil {
		return nil, err
	}

	resp, err := c.client.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	// parse response
}
```

---

## 三、给单次调用设置更短超时

```go
ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
defer cancel()

user, err := apiClient.GetUser(ctx, "alice")
```

这表示：

- 客户端断开，请求取消。
- 超过 2 秒，也取消。

---

## 四、区分 context 错误

```go
if err != nil {
	if errors.Is(err, context.DeadlineExceeded) {
		// timeout
	}
	if errors.Is(err, context.Canceled) {
		// canceled
	}
}
```

在 HTTP Client 中，错误可能被包装，实际项目中也可以通过日志记录原始错误，再对外返回统一错误。

---

## 五、本节检查点

请确认你能做到：

- 使用 `http.NewRequestWithContext` 创建请求。
- 在 Handler 中把 `r.Context()` 传给 Client。
- 用 `context.WithTimeout` 控制单次下游调用。
- 理解取消和超时的区别。

