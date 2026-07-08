# 1. r.Context 从哪里来

本节目标：理解每个 HTTP 请求自带的 context，以及为什么 Handler 中应该优先使用 `r.Context()`。

---

## 一、context 表达什么

`context.Context` 可以传递：

- 取消信号。
- 超时时间。
- 截止时间。
- 请求范围内的少量元信息。

在 HTTP 服务中，它最重要的意义是：

```text
这次请求是否还应该继续处理？
```

如果客户端断开连接，或者服务端决定取消请求，`r.Context()` 会被取消。

---

## 二、Handler 中的 context

每个请求都有：

```go
ctx := r.Context()
```

它来自标准库为当前请求创建的上下文。

你不需要自己从 `context.Background()` 创建请求 context。Handler 里应该优先使用 `r.Context()`，因为它和客户端连接、服务端关闭流程有关。

---

## 三、监听请求取消

示例：

```go
mux.HandleFunc("GET /wait", func(w http.ResponseWriter, r *http.Request) {
	select {
	case <-time.After(5 * time.Second):
		fmt.Fprintln(w, "done")
	case <-r.Context().Done():
		log.Printf("request canceled: %v", r.Context().Err())
	}
})
```

如果客户端在 5 秒内断开连接，`r.Context().Done()` 会被关闭。

---

## 四、不要丢掉请求 context

不推荐：

```go
func handler(w http.ResponseWriter, r *http.Request) {
	service.Do(context.Background())
}
```

这样下游任务不知道请求已经取消。

推荐：

```go
func handler(w http.ResponseWriter, r *http.Request) {
	service.Do(r.Context())
}
```

这样数据库、HTTP Client、缓存等下游调用都有机会及时停止。

---

## 五、在分层代码中传递 context

常见形式：

```go
func (h *Handler) CreateTodo(w http.ResponseWriter, r *http.Request) {
	todo, err := h.service.CreateTodo(r.Context(), title)
}
```

Service：

```go
func (s *Service) CreateTodo(ctx context.Context, title string) (Todo, error) {
	return s.store.Create(ctx, title)
}
```

Store：

```go
func (s *Store) Create(ctx context.Context, title string) (Todo, error) {
	// db.ExecContext(ctx, ...)
}
```

context 应该沿着调用链往下传。

---

## 六、本节检查点

请确认你能回答：

- Handler 中的 `r.Context()` 表示什么？
- 为什么不要用 `context.Background()` 替代请求 context？
- 客户端断开连接后 context 会发生什么？
- context 应该如何在 Handler、Service、Store 之间传递？

