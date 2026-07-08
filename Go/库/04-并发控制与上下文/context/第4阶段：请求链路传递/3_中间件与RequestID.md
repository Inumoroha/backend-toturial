# 3. 中间件与 Request ID

本节目标：学习如何通过中间件把 request id 放入 context，并在后续链路中读取。

`context.Value` 最典型的使用场景之一就是传递请求级元数据。

request id 就是一个好例子。

---

## 一、为什么需要 Request ID

一个请求可能经过很多层：

```text
handler -> service -> repository -> database -> downstream
```

如果日志里没有统一的 request id，很难把这些日志串起来。

有了 request id，可以在日志中看到：

```text
request_id=req-abc handler start
request_id=req-abc query user
request_id=req-abc call payment service
request_id=req-abc handler done
```

---

## 二、定义 context key

不要直接用 string 作为 key。

推荐：

```go
type contextKey string

const requestIDKey contextKey = "request_id"
```

这样可以减少和其他包 key 冲突的风险。

---

## 三、封装读写函数

```go
func WithRequestID(ctx context.Context, requestID string) context.Context {
	return context.WithValue(ctx, requestIDKey, requestID)
}

func RequestIDFromContext(ctx context.Context) (string, bool) {
	v, ok := ctx.Value(requestIDKey).(string)
	return v, ok
}
```

业务代码应该调用这些函数，而不是到处写：

```go
ctx.Value("request_id")
```

---

## 四、中间件示例

```go
func RequestIDMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		requestID := r.Header.Get("X-Request-ID")
		if requestID == "" {
			requestID = fmt.Sprintf("%d", time.Now().UnixNano())
		}

		ctx := WithRequestID(r.Context(), requestID)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}
```

关键点：

```go
r.WithContext(ctx)
```

它会返回一个带新 context 的请求副本。

---

## 五、在 service 中读取

```go
func (s *UserService) GetUser(ctx context.Context, id int64) (*User, error) {
	if requestID, ok := RequestIDFromContext(ctx); ok {
		fmt.Println("request id:", requestID)
	}

	return s.repo.FindByID(ctx, id)
}
```

实际项目里通常会把 request id 放到 logger 字段中。

---

## 六、本节练习

请完成：

1. 定义 `WithRequestID`。
2. 定义 `RequestIDFromContext`。
3. 写一个 HTTP middleware。
4. 从请求头 `X-Request-ID` 读取 request id。
5. 如果没有请求头，就生成一个简单 id。
6. 在 handler 中打印 request id。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 用自定义类型作为 context key。
- 封装 context value 的读写函数。
- 使用 `r.WithContext(ctx)` 更新请求 context。
- 理解 request id 属于适合放入 context 的请求级元数据。

---

## 八、完整中间件接入示例

```go
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
		requestID, _ := RequestIDFromContext(r.Context())
		fmt.Fprintf(w, "request_id=%s\n", requestID)
	})

	handler := RequestIDMiddleware(mux)

	fmt.Println("listen on :8080")
	_ = http.ListenAndServe(":8080", handler)
}
```

测试：

```powershell
curl -H "X-Request-ID: req-001" http://localhost:8080/hello
```

返回应该包含：

```text
request_id=req-001
```

---

## 九、为什么要用 r.WithContext

`r.Context()` 取出来的 ctx 是不可直接替换的。

你需要创建一个请求副本：

```go
next.ServeHTTP(w, r.WithContext(ctx))
```

如果你只写：

```go
ctx := WithRequestID(r.Context(), requestID)
next.ServeHTTP(w, r)
```

下游 handler 仍然拿不到新 ctx。

---

## 十、request id 生成方式

学习阶段可以用：

```go
fmt.Sprintf("%d", time.Now().UnixNano())
```

真实项目建议使用 UUID、ULID 或网关传入的 request id。

如果上游已经传入 `X-Request-ID`，通常应该沿用，方便跨服务排查。

---

## 十一、常见错误

### 1. key 使用普通字符串

容易和其他包冲突。

### 2. 直接强制类型断言

```go
ctx.Value(key).(string)
```

可能 panic。

### 3. 把业务 id 当 request id

request id 是请求唯一标识，不是用户 id、订单 id。

---

## 十二、本节练习

请实现两个中间件：

1. `RequestIDMiddleware`
2. `TraceIDMiddleware`

要求：

- 都从请求头读取。
- 没有就生成。
- 都写入 context。
- handler 中能同时读取 request id 和 trace id。
