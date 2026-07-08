# 5. 阶段练习：让 Todo 响应取消和超时

本节目标：把 context 用进 Todo API，模拟下游调用超时和客户端取消。

---

## 一、需求

在 Todo 服务中增加两个练习接口：

```text
GET /slow
GET /downstream
```

要求：

- `/slow` 模拟 10 秒任务，但客户端断开后服务端停止处理。
- `/downstream` 模拟调用下游服务，下游需要 3 秒，但当前接口只允许等待 2 秒。

---

## 二、实现 /slow

```go
mux.HandleFunc("GET /slow", func(w http.ResponseWriter, r *http.Request) {
	select {
	case <-time.After(10 * time.Second):
		writeJSON(w, http.StatusOK, map[string]string{"status": "done"})
	case <-r.Context().Done():
		log.Printf("slow request canceled: %v", r.Context().Err())
		return
	}
})
```

验证：

```bash
curl -i http://localhost:8080/slow
```

等待时按 `Ctrl+C`，观察服务端日志。

---

## 三、实现 /downstream

模拟函数：

```go
func fakeDownstream(ctx context.Context) error {
	select {
	case <-time.After(3 * time.Second):
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

Handler：

```go
mux.HandleFunc("GET /downstream", func(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
	defer cancel()

	if err := fakeDownstream(ctx); err != nil {
		writeError(w, http.StatusGatewayTimeout, "downstream_timeout", "downstream service timeout")
		return
	}

	writeJSON(w, http.StatusOK, map[string]string{"status": "ok"})
})
```

验证：

```bash
curl -i http://localhost:8080/downstream
```

应该等待约 2 秒后返回 `504`。

---

## 四、把 context 传入 service

如果你已经拆出了 service，可以把方法改成：

```go
func (s *TodoService) CreateTodo(ctx context.Context, title string) (Todo, error) {
	return s.store.Create(ctx, title)
}
```

Handler 中调用：

```go
todo, err := h.service.CreateTodo(r.Context(), req.Title)
```

即使当前内存 store 暂时用不到 context，也建议先保持这个签名，因为以后换成数据库时会自然接上。

---

## 五、阶段复盘

完成本阶段后，请写下：

- 哪些地方使用了 `r.Context()`？
- 哪些地方使用了 `context.WithTimeout`？
- 客户端取消和服务端超时有什么区别？
- 为什么下游调用不能无限等待？
- 哪些数据可以放到 `context.Value`？

如果这些能说清楚，就可以进入第 6 阶段：测试 `net/http` 服务。

