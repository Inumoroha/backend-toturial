# 3. 处理状态码和 JSON 响应

本节目标：学会正确处理 HTTP Client 的响应状态码和 JSON Body。

---

## 一、client.Do 不会因为 404 返回 error

这是非常重要的点：

```go
resp, err := client.Do(req)
```

只有网络错误、超时、请求构造错误等才会通过 `err` 返回。

如果服务端返回：

```text
404 Not Found
500 Internal Server Error
```

`err` 通常仍然是 `nil`，你需要自己检查：

```go
if resp.StatusCode < 200 || resp.StatusCode >= 300 {
	return fmt.Errorf("unexpected status: %d", resp.StatusCode)
}
```

---

## 二、解析 JSON

```go
var user User
if err := json.NewDecoder(resp.Body).Decode(&user); err != nil {
	return nil, err
}
```

结构体：

```go
type User struct {
	Login string `json:"login"`
	ID    int64  `json:"id"`
}
```

---

## 三、限制读取错误响应体

错误状态码时，你可能想读取响应体用于日志。但不要无限读：

```go
body, _ := io.ReadAll(io.LimitReader(resp.Body, 4<<10))
return fmt.Errorf("unexpected status %d: %s", resp.StatusCode, string(body))
```

`4<<10` 是 4KB。

---

## 四、封装错误类型

可以定义：

```go
type StatusError struct {
	StatusCode int
	Body       string
}

func (e *StatusError) Error() string {
	return fmt.Sprintf("unexpected status %d: %s", e.StatusCode, e.Body)
}
```

使用：

```go
if resp.StatusCode < 200 || resp.StatusCode >= 300 {
	body, _ := io.ReadAll(io.LimitReader(resp.Body, 4<<10))
	return nil, &StatusError{
		StatusCode: resp.StatusCode,
		Body:       string(body),
	}
}
```

这样调用方可以根据状态码做不同处理。

---

## 五、本节检查点

请确认你能回答：

- 为什么 404 不一定会让 `client.Do` 返回 error？
- 为什么非 2xx 状态码要自己判断？
- 为什么读取错误响应体时要限制大小？
- 如何定义一个包含状态码的错误类型？

