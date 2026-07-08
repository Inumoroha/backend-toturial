# 5. Context Value 综合练习

本节目标：完成一个 request id 和认证用户传递练习，巩固 `context.Value` 的正确用法。

---

## 一、练习目标

实现一个简单 HTTP 服务：

```text
GET /me
```

要求：

- RequestIDMiddleware 设置 request id。
- AuthMiddleware 模拟认证用户。
- Handler 从 context 读取认证用户。
- 日志函数从 context 读取 request id。
- userID 不通过 context 传业务参数。

---

## 二、定义 request context 工具

```go
type contextKey string

const (
	requestIDKey contextKey = "request_id"
	authUserKey  contextKey = "auth_user"
)

type AuthUser struct {
	ID    int64
	Email string
}

func WithRequestID(ctx context.Context, requestID string) context.Context {
	return context.WithValue(ctx, requestIDKey, requestID)
}

func RequestIDFromContext(ctx context.Context) (string, bool) {
	v, ok := ctx.Value(requestIDKey).(string)
	return v, ok
}

func WithAuthUser(ctx context.Context, user AuthUser) context.Context {
	return context.WithValue(ctx, authUserKey, user)
}

func AuthUserFromContext(ctx context.Context) (AuthUser, bool) {
	v, ok := ctx.Value(authUserKey).(AuthUser)
	return v, ok
}
```

---

## 三、中间件

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

func AuthMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		user := AuthUser{ID: 1001, Email: "tom@example.com"}
		ctx := WithAuthUser(r.Context(), user)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}
```

---

## 四、Handler

```go
func meHandler(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	user, ok := AuthUserFromContext(ctx)
	if !ok {
		http.Error(w, "unauthorized", http.StatusUnauthorized)
		return
	}

	logInfo(ctx, "get current user")
	_ = json.NewEncoder(w).Encode(user)
}

func logInfo(ctx context.Context, msg string) {
	requestID, _ := RequestIDFromContext(ctx)
	fmt.Printf("request_id=%s msg=%s\n", requestID, msg)
}
```

---

## 五、本阶段总结

你现在应该掌握：

- `context.Value` 适合请求级元数据。
- key 应该使用自定义类型。
- 读写 value 应该封装函数。
- request id、trace id、认证用户适合放 context。
- 业务参数、依赖对象、大对象不适合放 context。

下一阶段进入标准库和常用组件实践，把 context 放进更完整的后端组件里。

---

## 六、完整运行方式

创建项目：

```powershell
mkdir context-value-practice
cd context-value-practice
go mod init context-value-practice
```

创建：

```text
main.go
```

把本节代码补完整后运行：

```powershell
go run .
```

请求：

```powershell
curl -H "X-Request-ID: req-1" -H "Authorization: demo" http://localhost:8080/me
```

应该返回当前用户，并在日志中看到 request id。

---

## 七、main 函数示例

```go
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/me", meHandler)

	var handler http.Handler = mux
	handler = AuthMiddleware(handler)
	handler = RequestIDMiddleware(handler)

	fmt.Println("listen on :8080")
	_ = http.ListenAndServe(":8080", handler)
}
```

注意中间件顺序。

请求会先经过最外层的 `RequestIDMiddleware`，再经过 `AuthMiddleware`。

---

## 八、练习扩展：业务参数不要走 context

增加接口：

```text
GET /users/1001/orders?page=1
```

要求：

- 当前登录用户从 context 取。
- target user id 从 URL 解析。
- page 从 query 解析。
- request id 从 context 取。
- service 方法签名为：

```go
func (s *OrderService) ListOrders(ctx context.Context, targetUserID int64, page int) ([]Order, error)
```

这能帮助你区分：

```text
请求级元数据。
业务参数。
```

---

## 九、常见错误

### 1. 忘记 r.WithContext

中间件设置了 value，但 handler 读不到。

### 2. 认证用户不存在时继续处理

如果接口需要登录，读不到 auth user 应该返回 401。

### 3. request id 为空

没有请求头时应该生成一个。

### 4. key 用 string

练习阶段也尽量养成自定义 key 类型习惯。

---

## 十、本阶段复盘问题

1. request id 为什么适合放 context？
2. userID 查询参数为什么不适合放 context？
3. key 为什么不直接用 string？
4. 读取 context value 为什么要返回 `(value, bool)`？
5. 中间件为什么要调用 `r.WithContext(ctx)`？
