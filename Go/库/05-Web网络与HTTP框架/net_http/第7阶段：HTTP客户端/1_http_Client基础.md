# 1. http.Client 基础

本节目标：学会使用 `http.Client` 发送请求，并理解为什么它应该被复用。

---

## 一、最小 GET 请求

```go
resp, err := http.Get("https://api.github.com")
if err != nil {
	log.Fatal(err)
}
defer resp.Body.Close()

body, err := io.ReadAll(resp.Body)
if err != nil {
	log.Fatal(err)
}

fmt.Println(string(body))
```

这是最简单写法，但真实项目中更推荐显式使用 `http.Client` 和 `http.NewRequestWithContext`。

---

## 二、创建 Client

```go
client := &http.Client{
	Timeout: 5 * time.Second,
}
```

`Timeout` 是整个请求的超时，包括：

- 建立连接。
- 发送请求。
- 等待响应头。
- 读取响应体。

---

## 三、发送请求

```go
req, err := http.NewRequest(http.MethodGet, "https://api.github.com", nil)
if err != nil {
	log.Fatal(err)
}

resp, err := client.Do(req)
if err != nil {
	log.Fatal(err)
}
defer resp.Body.Close()
```

需要设置 Header 时：

```go
req.Header.Set("Accept", "application/json")
req.Header.Set("User-Agent", "net-http-learning")
```

---

## 四、为什么必须关闭响应体

每次请求成功拿到 `resp` 后，都应该：

```go
defer resp.Body.Close()
```

如果不关闭：

- 连接资源不能及时释放。
- 连接池复用受影响。
- 长期运行服务可能资源泄漏。

注意：只要 `err == nil` 且 `resp != nil`，就要关闭 Body，即使状态码是 404 或 500。

---

## 五、不要每次请求都新建 Client

不推荐：

```go
func callAPI() error {
	client := &http.Client{Timeout: 5 * time.Second}
	// use client
}
```

推荐：

```go
type APIClient struct {
	client *http.Client
}

func NewAPIClient() *APIClient {
	return &APIClient{
		client: &http.Client{Timeout: 5 * time.Second},
	}
}
```

`http.Client` 内部通过 Transport 管理连接复用。复用 Client 可以减少重复建连成本。

---

## 六、本节检查点

请确认你能回答：

- `http.Client` 的作用是什么？
- `client.Do(req)` 返回后为什么要关闭 Body？
- 为什么 `http.Client` 应该复用？
- `Client.Timeout` 大致覆盖哪些阶段？

