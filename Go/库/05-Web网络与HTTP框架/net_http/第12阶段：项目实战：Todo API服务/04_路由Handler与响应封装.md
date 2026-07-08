# 4. 路由、Handler 与响应封装

本节目标：实现 HTTP 层，负责路由注册、请求解析和响应输出。

---

## 一、Handler 结构体

```go
type Handler struct {
	service *todo.Service
}

func NewHandler(service *todo.Service) *Handler {
	return &Handler{service: service}
}
```

---

## 二、注册路由

```go
func (h *Handler) Routes() http.Handler {
	mux := http.NewServeMux()

	mux.HandleFunc("GET /health", h.health)
	mux.HandleFunc("GET /todos", h.listTodos)
	mux.HandleFunc("POST /todos", h.createTodo)
	mux.HandleFunc("GET /todos/{id}", h.getTodo)
	mux.HandleFunc("PATCH /todos/{id}", h.updateTodo)
	mux.HandleFunc("DELETE /todos/{id}", h.deleteTodo)

	return mux
}
```

---

## 三、响应封装

```go
func writeJSON(w http.ResponseWriter, status int, data any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	_ = json.NewEncoder(w).Encode(data)
}

func writeError(w http.ResponseWriter, status int, code, message string) {
	writeJSON(w, status, map[string]any{
		"error": map[string]string{
			"code":    code,
			"message": message,
		},
	})
}
```

---

## 四、解析 ID

```go
func parseID(r *http.Request) (int, error) {
	id, err := strconv.Atoi(r.PathValue("id"))
	if err != nil || id <= 0 {
		return 0, fmt.Errorf("invalid id")
	}
	return id, nil
}
```

---

## 五、Handler 原则

Handler 只做：

- 读取请求。
- 校验 HTTP 层参数。
- 调用 service。
- 把错误映射成 HTTP 响应。
- 写响应。

不要在 Handler 里写大量业务规则。

---

## 六、本节检查点

请确认你能回答：

- `Routes()` 为什么返回 `http.Handler`？
- `writeJSON` 和 `writeError` 解决什么重复？
- Handler 和 Service 的边界是什么？

