# 2. 日志、耗时与 Request ID 中间件

本节目标：实现三个常用中间件能力：请求日志、耗时统计、Request ID。

---

## 一、为什么需要请求日志

线上排查问题时，你经常需要知道：

- 谁访问了哪个接口。
- 请求方法是什么。
- 响应状态码是多少。
- 请求耗时多久。
- 客户端地址是什么。
- 某条业务日志属于哪个请求。

没有请求日志，服务一旦出问题，就很难还原现场。

---

## 二、最小日志中间件

```go
func logging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		log.Printf("%s %s %s", r.Method, r.URL.Path, time.Since(start))
	})
}
```

导入：

```go
import "time"
```

这个版本能记录方法、路径、耗时，但还拿不到响应状态码。

---

## 三、捕获响应状态码

`http.ResponseWriter` 本身没有提供“读取已写状态码”的方法。常见做法是包装它。

```go
type statusRecorder struct {
	http.ResponseWriter
	status int
}

func (r *statusRecorder) WriteHeader(status int) {
	r.status = status
	r.ResponseWriter.WriteHeader(status)
}
```

使用：

```go
func logging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		rec := &statusRecorder{
			ResponseWriter: w,
			status:         http.StatusOK,
		}

		next.ServeHTTP(rec, r)

		log.Printf(
			"method=%s path=%s status=%d duration=%s remote_addr=%s",
			r.Method,
			r.URL.Path,
			rec.status,
			time.Since(start),
			r.RemoteAddr,
		)
	})
}
```

为什么默认是 `200`？

因为如果 Handler 直接写 Body 而没有调用 `WriteHeader`，Go 会默认返回 `200 OK`。

---

## 四、Request ID 的作用

一个请求可能产生多条日志：

```text
收到请求
解析参数
调用数据库
返回响应
```

如果没有统一 ID，很难知道这些日志是否属于同一个请求。

Request ID 可以放在：

- 请求 context。
- 响应 Header。
- 日志字段。

---

## 五、实现简单 Request ID

先定义 context key：

```go
type contextKey string

const requestIDKey contextKey = "request_id"
```

生成简单 ID：

```go
func newRequestID() string {
	return fmt.Sprintf("%d", time.Now().UnixNano())
}
```

中间件：

```go
func requestID(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		id := r.Header.Get("X-Request-ID")
		if id == "" {
			id = newRequestID()
		}

		w.Header().Set("X-Request-ID", id)

		ctx := context.WithValue(r.Context(), requestIDKey, id)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}
```

读取：

```go
func getRequestID(r *http.Request) string {
	id, _ := r.Context().Value(requestIDKey).(string)
	return id
}
```

---

## 六、把 Request ID 打进日志

```go
func logging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		rec := &statusRecorder{ResponseWriter: w, status: http.StatusOK}

		next.ServeHTTP(rec, r)

		log.Printf(
			"request_id=%s method=%s path=%s status=%d duration=%s",
			getRequestID(r),
			r.Method,
			r.URL.Path,
			rec.status,
			time.Since(start),
		)
	})
}
```

注意：中间件顺序要让 `requestID` 在 `logging` 之前执行，否则 logging 读不到 ID。

```go
handler := chain(mux, requestID, logging)
```

---

## 七、关于 context.Value 的克制

`context.Value` 不应该用来传普通业务参数。它适合放请求范围内的元信息，例如：

- Request ID。
- 当前用户 ID。
- Trace ID。

不要把大量业务对象塞进 context，这会让代码依赖关系变得隐晦。

---

## 八、本节检查点

请确认你能做到：

- 写一个记录耗时的日志中间件。
- 包装 `ResponseWriter` 捕获状态码。
- 生成或读取 `X-Request-ID`。
- 把 Request ID 放入 context。
- 解释为什么中间件顺序会影响日志内容。

