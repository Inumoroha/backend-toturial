# 3. 项目一：超时错误与 Request ID 日志

本节目标：给聚合服务补上 request id、错误分类和日志。

---

## 一、Request ID 工具

```go
type contextKey string

const requestIDKey contextKey = "request_id"

func WithRequestID(ctx context.Context, requestID string) context.Context {
	return context.WithValue(ctx, requestIDKey, requestID)
}

func RequestIDFromContext(ctx context.Context) (string, bool) {
	v, ok := ctx.Value(requestIDKey).(string)
	return v, ok
}
```

---

## 二、中间件

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

---

## 三、日志函数

```go
func logInfo(ctx context.Context, msg string) {
	requestID, _ := RequestIDFromContext(ctx)
	fmt.Printf("level=info request_id=%s msg=%s\n", requestID, msg)
}

func logError(ctx context.Context, msg string, err error) {
	requestID, _ := RequestIDFromContext(ctx)
	fmt.Printf("level=error request_id=%s msg=%s err=%v\n", requestID, msg, err)
}
```

---

## 四、错误分类响应

```go
profile, err := h.service.GetProfile(ctx, userID)
if err != nil {
	logError(ctx, "get profile failed", err)

	if errors.Is(err, context.DeadlineExceeded) {
		http.Error(w, "profile timeout", http.StatusGatewayTimeout)
		return
	}

	if errors.Is(err, context.Canceled) {
		http.Error(w, "request canceled", http.StatusRequestTimeout)
		return
	}

	http.Error(w, "bad gateway", http.StatusBadGateway)
	return
}
```

---

## 五、下游错误要带名称

不推荐：

```go
return ctx.Err()
```

推荐：

```go
return fmt.Errorf("fetch wallet stopped: %w", ctx.Err())
```

这样日志里能看出哪个下游出了问题，同时还保留 context 错误根因。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 给请求注入 request id。
- 在日志中打印 request id。
- 区分超时、取消和普通下游错误。
- 给下游错误添加清楚的业务上下文。

---

## 七、中间件接入 main

```go
mux := http.NewServeMux()
mux.HandleFunc("/profile/1001", handler.GetProfile)

wrapped := RequestIDMiddleware(mux)

server := &http.Server{
	Addr:    ":8080",
	Handler: wrapped,
}
```

这样所有请求都会先经过 RequestIDMiddleware。

测试：

```powershell
curl -H "X-Request-ID: req-demo-1" http://localhost:8080/profile/1001
```

日志中应该能看到：

```text
request_id=req-demo-1
```

---

## 八、日志位置建议

建议在这些位置打日志：

```text
handler 开始。
service 开始聚合。
每个下游开始和结束。
发生错误时。
handler 返回响应前。
```

不要每行代码都打日志，但关键边界要有。

例如：

```go
func fetchUser(ctx context.Context, userID int64) (User, error) {
	start := time.Now()
	logInfo(ctx, "fetch user start")
	defer func() {
		logInfo(ctx, fmt.Sprintf("fetch user done cost=%s", time.Since(start)))
	}()

	select {
	case <-time.After(100 * time.Millisecond):
		return User{ID: userID, Name: "Tom"}, nil
	case <-ctx.Done():
		return User{}, fmt.Errorf("fetch user stopped: %w", ctx.Err())
	}
}
```

---

## 九、错误响应策略

推荐策略：

| 错误 | HTTP 状态码 | 说明 |
| --- | --- | --- |
| `context.DeadlineExceeded` | 504 | 请求或下游超时 |
| `context.Canceled` | 408 或直接停止 | 客户端或上游取消 |
| 下游普通错误 | 502 | 依赖失败 |
| 参数错误 | 400 | 用户输入问题 |
| 未知错误 | 500 | 服务内部错误 |

不同团队可以调整状态码，但必须保持一致。

---

## 十、错误包装格式

下游错误建议带操作名：

```go
return fmt.Errorf("fetch orders for user %d: %w", userID, err)
```

context 取消也一样：

```go
return fmt.Errorf("fetch orders for user %d stopped: %w", userID, ctx.Err())
```

这样上层既能判断：

```go
errors.Is(err, context.DeadlineExceeded)
```

又能看懂日志：

```text
fetch orders for user 1001 stopped: context deadline exceeded
```

---

## 十一、练习：日志排查

请人为制造一个超时：

```text
fetchOrders 耗时 2s。
总超时 800ms。
```

然后观察：

1. 日志里是否有 request id。
2. 日志里是否能看出哪个下游超时。
3. HTTP 状态码是否是 504。
4. 错误是否保留 `DeadlineExceeded` 根因。
