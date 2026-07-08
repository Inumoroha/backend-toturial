# 8. 实现 httpapi：响应封装与 Handler

本节目标：实现 HTTP 层，把请求转换成业务调用，再把业务结果转换成 HTTP 响应。

---

## 一、response.go

文件：`internal/httpapi/response.go`

```go
package httpapi

import (
	"encoding/json"
	"net/http"
)

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

所有 JSON 响应都走这两个函数，项目风格会稳定很多。

---

## 二、handler.go

文件：`internal/httpapi/handler.go`

```go
package httpapi

import (
	"encoding/json"
	"errors"
	"net/http"
	"strconv"

	"example.com/todo-api/internal/todo"
)

type Handler struct {
	service *todo.Service
}

func NewHandler(service *todo.Service) *Handler {
	return &Handler{service: service}
}
```

如果你的 module 不是 `example.com/todo-api`，导入路径要改成你自己的 module 名称。

---

## 三、routes.go

文件：`internal/httpapi/routes.go`

```go
package httpapi

import "net/http"

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

## 四、基础 Handler

```go
func (h *Handler) health(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusOK, map[string]string{"status": "ok"})
}

func (h *Handler) listTodos(w http.ResponseWriter, r *http.Request) {
	items, err := h.service.List(r.Context())
	if err != nil {
		writeError(w, http.StatusInternalServerError, "internal_error", "internal server error")
		return
	}
	writeJSON(w, http.StatusOK, map[string]any{"items": items})
}
```

---

## 五、创建 Todo

```go
type createTodoRequest struct {
	Title string `json:"title"`
}

func (h *Handler) createTodo(w http.ResponseWriter, r *http.Request) {
	r.Body = http.MaxBytesReader(w, r.Body, 1<<20)

	var req createTodoRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		writeError(w, http.StatusBadRequest, "invalid_json", "invalid json body")
		return
	}

	item, err := h.service.Create(r.Context(), req.Title)
	if err != nil {
		h.writeTodoError(w, err)
		return
	}
	writeJSON(w, http.StatusCreated, item)
}
```

---

## 六、解析 ID 和错误映射

```go
func parseID(r *http.Request) (int, error) {
	id, err := strconv.Atoi(r.PathValue("id"))
	if err != nil || id <= 0 {
		return 0, errors.New("invalid id")
	}
	return id, nil
}

func (h *Handler) writeTodoError(w http.ResponseWriter, err error) {
	switch {
	case errors.Is(err, todo.ErrInvalidTitle):
		writeError(w, http.StatusBadRequest, "invalid_request", "title is required")
	case errors.Is(err, todo.ErrNotFound):
		writeError(w, http.StatusNotFound, "not_found", "todo not found")
	default:
		writeError(w, http.StatusInternalServerError, "internal_error", "internal server error")
	}
}
```

---

## 七、详情、更新、删除

实现思路：

```text
getTodo:
  parseID -> service.Get -> writeJSON

updateTodo:
  parseID -> decode JSON -> service.Update -> writeJSON

deleteTodo:
  parseID -> service.Delete -> WriteHeader(204)
```

删除成功不要再写 JSON Body：

```go
w.WriteHeader(http.StatusNoContent)
```

---

## 八、本节检查点

完成后运行：

```bash
go test ./...
```

你应该能解释：

- HTTP 层为什么要限制 Body。
- Handler 如何把业务错误映射成状态码。
- 为什么删除成功返回 `204`。
- 为什么 `Routes()` 返回 `http.Handler`。
