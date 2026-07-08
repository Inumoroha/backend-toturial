# 5. 阶段练习：给 Todo 加入中间件

本节目标：把第 2 阶段的内存 Todo API 改造成更接近真实工程的版本，加入日志、Request ID、Recovery、鉴权和 CORS。

---

## 一、需求

在 Todo API 基础上增加：

- 所有请求生成或透传 `X-Request-ID`。
- 每个请求打印日志：方法、路径、状态码、耗时、Request ID。
- Handler panic 时返回统一 JSON `500`。
- 写操作需要 Token。
- 支持本地前端跨域访问。

写操作包括：

```text
POST   /todos
PATCH  /todos/{id}
DELETE /todos/{id}
```

读操作不需要鉴权：

```text
GET /todos
GET /todos/{id}
```

---

## 二、中间件类型和 chain

```go
type Middleware func(http.Handler) http.Handler

func chain(h http.Handler, middlewares ...Middleware) http.Handler {
	for i := len(middlewares) - 1; i >= 0; i-- {
		h = middlewares[i](h)
	}
	return h
}
```

---

## 三、注册路由时区分公开和私有

```go
mux := http.NewServeMux()

mux.HandleFunc("GET /todos", store.listTodos)
mux.HandleFunc("GET /todos/{id}", store.getTodo)

mux.Handle("POST /todos", auth(http.HandlerFunc(store.createTodo)))
mux.Handle("PATCH /todos/{id}", auth(http.HandlerFunc(store.updateTodo)))
mux.Handle("DELETE /todos/{id}", auth(http.HandlerFunc(store.deleteTodo)))
```

因为 `auth` 返回 `http.Handler`，这里使用 `mux.Handle`。

---

## 四、全局包装基础中间件

```go
handler := chain(
	mux,
	recovery,
	requestID,
	logging,
	cors,
)

log.Fatal(http.ListenAndServe(":8080", handler))
```

建议顺序：

- `recovery` 放外层，尽量兜住后续 panic。
- `requestID` 尽早生成，后续日志可以使用。
- `logging` 包住业务处理，记录最终状态和耗时。
- `cors` 处理跨域 Header 和 OPTIONS。

---

## 五、验证命令

公开读：

```bash
curl -i http://localhost:8080/todos
```

未授权创建：

```bash
curl -i -X POST http://localhost:8080/todos `
  -H "Content-Type: application/json" `
  -d "{\"title\":\"learn middleware\"}"
```

应该返回 `401`。

授权创建：

```bash
curl -i -X POST http://localhost:8080/todos `
  -H "Content-Type: application/json" `
  -H "Authorization: Bearer dev-token" `
  -H "X-Request-ID: test-001" `
  -d "{\"title\":\"learn middleware\"}"
```

观察：

- 响应 Header 是否有 `X-Request-ID`。
- 服务端日志是否打印 `test-001`。
- 状态码是否是 `201`。

---

## 六、故意测试 panic

临时加一个路由：

```go
mux.HandleFunc("GET /panic", func(w http.ResponseWriter, r *http.Request) {
	panic("boom")
})
```

访问：

```bash
curl -i http://localhost:8080/panic
```

应该返回统一 JSON 错误：

```json
{
  "error": {
    "code": "internal_error",
    "message": "internal server error"
  }
}
```

测试完可以删除这个路由。

---

## 七、阶段复盘

完成练习后，请确认你能解释：

- 为什么中间件适合处理日志和鉴权？
- 为什么写操作需要鉴权，读操作可以公开？
- 为什么 `auth` 包装路由时要使用 `mux.Handle`？
- Request ID 在日志排查中有什么价值？
- Recovery 能兜底什么，不能替代什么？

如果这些问题都清楚，就可以进入第 4 阶段：显式配置 `http.Server`、超时和优雅关闭。

