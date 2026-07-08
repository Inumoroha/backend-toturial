# 1. Query、路径参数与方法判断

本节目标：掌握三类最常见的请求信息读取方式：Query 参数、Path 参数和 Method。

---

## 一、读取 Query 参数

Query 参数适合表达过滤、搜索、分页等条件。

URL：

```text
/todos?done=false&page=1
```

读取：

```go
done := r.URL.Query().Get("done")
page := r.URL.Query().Get("page")
```

示例：

```go
mux.HandleFunc("GET /todos", func(w http.ResponseWriter, r *http.Request) {
	done := r.URL.Query().Get("done")
	page := r.URL.Query().Get("page")

	fmt.Fprintf(w, "done=%s page=%s\n", done, page)
})
```

验证：

```bash
curl -i "http://localhost:8080/todos?done=false&page=1"
```

---

## 二、给 Query 设置默认值

Query 参数可能不存在。比如分页接口可以给默认值：

```go
page := r.URL.Query().Get("page")
if page == "" {
	page = "1"
}
```

如果需要整数，要转换：

```go
page, err := strconv.Atoi(r.URL.Query().Get("page"))
if err != nil || page <= 0 {
	http.Error(w, "invalid page", http.StatusBadRequest)
	return
}
```

注意：不要相信客户端传来的参数。只要来自请求，就需要校验。

---

## 三、读取路径参数

Go 1.22 以后：

```go
mux.HandleFunc("GET /todos/{id}", func(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")
	fmt.Fprintf(w, "todo id=%s\n", id)
})
```

访问：

```bash
curl -i http://localhost:8080/todos/10
```

路径参数适合表示资源身份：

```text
/users/{id}
/orders/{id}
/articles/{slug}
```

---

## 四、把路径参数转成整数

很多 ID 是整数：

```go
idText := r.PathValue("id")
id, err := strconv.Atoi(idText)
if err != nil || id <= 0 {
	http.Error(w, "invalid id", http.StatusBadRequest)
	return
}
```

这里返回 `400`，因为客户端传了非法参数。

如果 ID 格式正确，但资源不存在，应该返回 `404`：

```text
/todos/abc -> 400
/todos/999 -> 404
```

这两个状态码表达的含义不同。

---

## 五、方法判断

如果使用 Go 1.22 的路由模式：

```go
mux.HandleFunc("GET /todos", listTodos)
mux.HandleFunc("POST /todos", createTodo)
```

通常不需要在 Handler 里手动判断方法。

如果使用旧写法：

```go
mux.HandleFunc("/todos", func(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodGet {
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}

	fmt.Fprintln(w, "list todos")
})
```

---

## 六、常见设计错误

### 1. 把资源 ID 放 Query

不推荐：

```text
GET /todos?id=1
```

更推荐：

```text
GET /todos/1
```

### 2. 用 Query 表达复杂创建数据

不推荐：

```text
POST /todos?title=learn&done=false
```

更推荐 JSON Body：

```json
{"title":"learn","done":false}
```

### 3. 不区分 400 和 404

```text
/todos/abc -> 400，因为 id 格式错误。
/todos/100 -> 404，因为 id 格式正确但资源不存在。
```

---

## 七、本节检查点

请确认你能做到：

- 读取 Query 参数。
- 给 Query 参数设置默认值。
- 读取 Path 参数。
- 把 Path 参数转成整数并校验。
- 解释什么时候返回 `400`，什么时候返回 `404`。

