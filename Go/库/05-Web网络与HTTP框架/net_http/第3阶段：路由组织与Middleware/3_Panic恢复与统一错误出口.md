# 3. Panic 恢复与统一错误出口

本节目标：学会用 recovery 中间件捕获 Handler 中的 panic，避免单个请求的异常影响服务稳定性。

---

## 一、panic 在 HTTP 服务里意味着什么

如果 Handler 中发生 panic：

```go
mux.HandleFunc("GET /panic", func(w http.ResponseWriter, r *http.Request) {
	panic("boom")
})
```

标准库会保护连接处理 goroutine，不至于让整个进程直接退出，但客户端可能得到不友好的响应，日志也不够统一。

生产服务通常会自己加 recovery 中间件：

- 捕获 panic。
- 记录日志。
- 返回统一 `500` 响应。

---

## 二、最小 recovery 中间件

```go
func recovery(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if rec := recover(); rec != nil {
				log.Printf("panic: %v", rec)
				http.Error(w, "internal server error", http.StatusInternalServerError)
			}
		}()

		next.ServeHTTP(w, r)
	})
}
```

使用：

```go
handler := recovery(mux)
http.ListenAndServe(":8080", handler)
```

---

## 三、返回 JSON 错误

如果项目是 JSON API，推荐：

```go
func recovery(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if rec := recover(); rec != nil {
				log.Printf("panic: %v", rec)
				writeError(w, http.StatusInternalServerError, "internal_error", "internal server error")
			}
		}()

		next.ServeHTTP(w, r)
	})
}
```

不要把 panic 内容直接返回给客户端：

```json
{"error":"runtime error: invalid memory address or nil pointer dereference"}
```

这会暴露内部实现细节。

---

## 四、什么时候不该用 panic

不要把普通业务错误写成 panic。

不推荐：

```go
if title == "" {
	panic("title required")
}
```

推荐：

```go
if title == "" {
	writeError(w, http.StatusBadRequest, "invalid_request", "title is required")
	return
}
```

panic 更适合表示程序员错误或不可恢复异常，而不是常规业务分支。

---

## 五、已经写出响应后再 panic 怎么办

如果 Handler 已经写了部分响应：

```go
fmt.Fprintln(w, "hello")
panic("boom")
```

这时响应头可能已经发送，recovery 再写 `500` 未必能改变客户端看到的状态码。

所以 recovery 不是万能的。它主要用于兜底，不能替代正常错误处理。

---

## 六、统一错误出口的意义

中间件负责兜底错误：

- panic。
- 未认证。
- CORS 预检。
- 请求体过大。

Handler 负责业务错误：

- 参数错误。
- 资源不存在。
- 状态冲突。
- 创建失败。

两者配合，接口风格才稳定。

---

## 七、本节检查点

请确认你能回答：

- recovery 中间件为什么要使用 `defer`？
- `recover()` 能捕获什么？
- 为什么不要把 panic 内容返回给客户端？
- 为什么普通参数错误不应该 panic？
- 已经写出响应后 panic 有什么问题？

