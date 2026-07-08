# 2. JSON 请求与响应

本节目标：学会读取 JSON 请求体，并返回 JSON 响应。JSON API 是 Go 后端最常见的工作内容之一。

---

## 一、定义请求结构体

假设创建 Todo 的请求长这样：

```json
{
  "title": "learn net/http"
}
```

可以定义：

```go
type createTodoRequest struct {
	Title string `json:"title"`
}
```

结构体字段要导出，也就是首字母大写，否则 `encoding/json` 无法设置它。

---

## 二、读取 JSON Body

Handler 示例：

```go
func createTodo(w http.ResponseWriter, r *http.Request) {
	var req createTodoRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "invalid json", http.StatusBadRequest)
		return
	}

	fmt.Fprintf(w, "title=%s\n", req.Title)
}
```

需要导入：

```go
import "encoding/json"
```

验证：

```bash
curl -i -X POST http://localhost:8080/todos `
  -H "Content-Type: application/json" `
  -d "{\"title\":\"learn net/http\"}"
```

---

## 三、校验请求字段

JSON 格式正确，不代表业务参数正确。

例如：

```json
{"title":""}
```

应该返回 `400`：

```go
if strings.TrimSpace(req.Title) == "" {
	http.Error(w, "title is required", http.StatusBadRequest)
	return
}
```

导入：

```go
import "strings"
```

后端不能指望前端一定传正确。前端校验是用户体验，后端校验是系统边界。

---

## 四、返回 JSON

响应结构体：

```go
type todoResponse struct {
	ID    int    `json:"id"`
	Title string `json:"title"`
	Done  bool   `json:"done"`
}
```

写响应：

```go
w.Header().Set("Content-Type", "application/json")
w.WriteHeader(http.StatusCreated)
json.NewEncoder(w).Encode(todoResponse{
	ID:    1,
	Title: req.Title,
	Done:  false,
})
```

注意顺序：

```text
先设置 Header
再设置状态码
最后写 Body
```

---

## 五、封装 writeJSON

每个 Handler 都手写会重复。可以封装：

```go
func writeJSON(w http.ResponseWriter, status int, data any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	_ = json.NewEncoder(w).Encode(data)
}
```

使用：

```go
writeJSON(w, http.StatusCreated, todoResponse{
	ID:    1,
	Title: req.Title,
	Done:  false,
})
```

这里暂时忽略 Encode 错误，因为写响应失败通常意味着客户端断开连接，后面讲日志和观测时再细化。

---

## 六、限制请求体大小

不要让客户端无限制上传 Body。即使是 JSON，也应该限制大小。

示例：

```go
r.Body = http.MaxBytesReader(w, r.Body, 1<<20) // 1MB
```

放在 Decode 前：

```go
func createTodo(w http.ResponseWriter, r *http.Request) {
	r.Body = http.MaxBytesReader(w, r.Body, 1<<20)

	var req createTodoRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "invalid json", http.StatusBadRequest)
		return
	}
}
```

这是安全基础。真实服务不要无限读请求体。

---

## 七、防止多个 JSON 值

下面这种请求其实包含两个 JSON 值：

```json
{"title":"a"}{"title":"b"}
```

简单 `Decode` 只会读第一个。更严谨的做法是 Decode 后再检查是否还有额外内容。初学阶段可以先知道这个问题，项目阶段再封装更严格的 JSON 解析函数。

---

## 八、本节完整示例

```go
package main

import (
	"encoding/json"
	"log"
	"net/http"
	"strings"
)

type createTodoRequest struct {
	Title string `json:"title"`
}

type todoResponse struct {
	ID    int    `json:"id"`
	Title string `json:"title"`
	Done  bool   `json:"done"`
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("POST /todos", createTodo)

	log.Println("server listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}

func createTodo(w http.ResponseWriter, r *http.Request) {
	r.Body = http.MaxBytesReader(w, r.Body, 1<<20)

	var req createTodoRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "invalid json", http.StatusBadRequest)
		return
	}

	if strings.TrimSpace(req.Title) == "" {
		http.Error(w, "title is required", http.StatusBadRequest)
		return
	}

	writeJSON(w, http.StatusCreated, todoResponse{
		ID:    1,
		Title: req.Title,
		Done:  false,
	})
}

func writeJSON(w http.ResponseWriter, status int, data any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	_ = json.NewEncoder(w).Encode(data)
}
```

---

## 九、本节检查点

请确认你能回答：

- 为什么结构体字段要大写？
- `json.NewDecoder(r.Body).Decode(&req)` 做了什么？
- 为什么 JSON 格式正确后还要做业务校验？
- 为什么写 JSON 前要设置 `Content-Type`？
- 为什么要限制请求体大小？

