# 5. 阶段练习：内存 Todo JSON API

本节目标：实现一个不依赖数据库的 Todo API，把 JSON 请求、JSON 响应、路径参数、状态码和错误处理串起来。

---

## 一、需求

实现接口：

```text
GET    /todos
POST   /todos
GET    /todos/{id}
PATCH  /todos/{id}
DELETE /todos/{id}
```

数据先保存在内存 map 中。

Todo 字段：

```go
type Todo struct {
	ID    int    `json:"id"`
	Title string `json:"title"`
	Done  bool   `json:"done"`
}
```

---

## 二、项目结构建议

本阶段可以先用单文件：

```text
playground/
  go.mod
  main.go
```

不要急着拆目录。先把 HTTP 行为练清楚。

---

## 三、完整参考实现

```go
package main

import (
	"encoding/json"
	"log"
	"net/http"
	"strconv"
	"strings"
	"sync"
)

type Todo struct {
	ID    int    `json:"id"`
	Title string `json:"title"`
	Done  bool   `json:"done"`
}

type createTodoRequest struct {
	Title string `json:"title"`
}

type updateTodoRequest struct {
	Title *string `json:"title"`
	Done  *bool   `json:"done"`
}

type todoStore struct {
	mu     sync.RWMutex
	nextID int
	items  map[int]Todo
}

func newTodoStore() *todoStore {
	return &todoStore{
		nextID: 1,
		items:  make(map[int]Todo),
	}
}

func main() {
	store := newTodoStore()

	mux := http.NewServeMux()
	mux.HandleFunc("GET /todos", store.listTodos)
	mux.HandleFunc("POST /todos", store.createTodo)
	mux.HandleFunc("GET /todos/{id}", store.getTodo)
	mux.HandleFunc("PATCH /todos/{id}", store.updateTodo)
	mux.HandleFunc("DELETE /todos/{id}", store.deleteTodo)

	log.Println("server listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}

func (s *todoStore) listTodos(w http.ResponseWriter, r *http.Request) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	items := make([]Todo, 0, len(s.items))
	for _, item := range s.items {
		items = append(items, item)
	}

	writeJSON(w, http.StatusOK, map[string]any{"items": items})
}

func (s *todoStore) createTodo(w http.ResponseWriter, r *http.Request) {
	var req createTodoRequest
	if err := readJSON(w, r, &req); err != nil {
		writeError(w, http.StatusBadRequest, "invalid_json", "invalid json body")
		return
	}

	req.Title = strings.TrimSpace(req.Title)
	if req.Title == "" {
		writeError(w, http.StatusBadRequest, "invalid_request", "title is required")
		return
	}

	s.mu.Lock()
	defer s.mu.Unlock()

	todo := Todo{
		ID:    s.nextID,
		Title: req.Title,
		Done:  false,
	}
	s.items[todo.ID] = todo
	s.nextID++

	writeJSON(w, http.StatusCreated, todo)
}

func (s *todoStore) getTodo(w http.ResponseWriter, r *http.Request) {
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	s.mu.RLock()
	defer s.mu.RUnlock()

	todo, ok := s.items[id]
	if !ok {
		writeError(w, http.StatusNotFound, "not_found", "todo not found")
		return
	}

	writeJSON(w, http.StatusOK, todo)
}

func (s *todoStore) updateTodo(w http.ResponseWriter, r *http.Request) {
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	var req updateTodoRequest
	if err := readJSON(w, r, &req); err != nil {
		writeError(w, http.StatusBadRequest, "invalid_json", "invalid json body")
		return
	}

	s.mu.Lock()
	defer s.mu.Unlock()

	todo, ok := s.items[id]
	if !ok {
		writeError(w, http.StatusNotFound, "not_found", "todo not found")
		return
	}

	if req.Title != nil {
		title := strings.TrimSpace(*req.Title)
		if title == "" {
			writeError(w, http.StatusBadRequest, "invalid_request", "title cannot be empty")
			return
		}
		todo.Title = title
	}

	if req.Done != nil {
		todo.Done = *req.Done
	}

	s.items[id] = todo
	writeJSON(w, http.StatusOK, todo)
}

func (s *todoStore) deleteTodo(w http.ResponseWriter, r *http.Request) {
	id, ok := parseID(w, r)
	if !ok {
		return
	}

	s.mu.Lock()
	defer s.mu.Unlock()

	if _, ok := s.items[id]; !ok {
		writeError(w, http.StatusNotFound, "not_found", "todo not found")
		return
	}

	delete(s.items, id)
	w.WriteHeader(http.StatusNoContent)
}

func parseID(w http.ResponseWriter, r *http.Request) (int, bool) {
	id, err := strconv.Atoi(r.PathValue("id"))
	if err != nil || id <= 0 {
		writeError(w, http.StatusBadRequest, "invalid_id", "invalid id")
		return 0, false
	}

	return id, true
}

func readJSON(w http.ResponseWriter, r *http.Request, dst any) error {
	r.Body = http.MaxBytesReader(w, r.Body, 1<<20)
	return json.NewDecoder(r.Body).Decode(dst)
}

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

## 四、为什么内存 map 要加锁

Go 的普通 map 不能并发读写。HTTP 服务中，不同请求会由不同 goroutine 处理。

如果两个请求同时操作：

```go
items[id] = todo
delete(items, id)
```

就可能发生 data race，甚至直接 panic。

所以这里使用：

```go
sync.RWMutex
```

- 读操作用 `RLock`。
- 写操作用 `Lock`。

后面性能与并发阶段会更深入讲。

---

## 五、验证命令

创建：

```bash
curl -i -X POST http://localhost:8080/todos `
  -H "Content-Type: application/json" `
  -d "{\"title\":\"learn net/http\"}"
```

列表：

```bash
curl -i http://localhost:8080/todos
```

详情：

```bash
curl -i http://localhost:8080/todos/1
```

更新：

```bash
curl -i -X PATCH http://localhost:8080/todos/1 `
  -H "Content-Type: application/json" `
  -d "{\"done\":true}"
```

删除：

```bash
curl -i -X DELETE http://localhost:8080/todos/1
```

错误场景：

```bash
curl -i -X POST http://localhost:8080/todos -H "Content-Type: application/json" -d "{}"
curl -i http://localhost:8080/todos/abc
curl -i http://localhost:8080/todos/999
```

---

## 六、本阶段复盘

完成练习后，请确认你能解释：

- 为什么创建成功返回 `201`？
- 为什么删除成功可以返回 `204`？
- 为什么 `PATCH` 请求中字段使用指针？
- 为什么内存 map 要加锁？
- `parseID`、`writeJSON`、`writeError` 分别解决了什么重复问题？

如果这些都能说清楚，就可以进入第 3 阶段：路由组织与 Middleware。

