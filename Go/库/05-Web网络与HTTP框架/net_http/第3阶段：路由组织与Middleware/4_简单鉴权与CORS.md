# 4. 简单鉴权与 CORS

本节目标：实现两个常见中间件：Bearer Token 鉴权和 CORS。它们都是 HTTP 服务中经常出现的横切逻辑。

---

## 一、简单 Token 鉴权

学习阶段可以用固定 Token：

```text
Authorization: Bearer dev-token
```

中间件：

```go
func auth(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.Header.Get("Authorization") != "Bearer dev-token" {
			writeError(w, http.StatusUnauthorized, "unauthorized", "missing or invalid token")
			return
		}

		next.ServeHTTP(w, r)
	})
}
```

验证：

```bash
curl -i http://localhost:8080/private
curl -i http://localhost:8080/private -H "Authorization: Bearer dev-token"
```

---

## 二、只保护部分路由

不是所有接口都需要鉴权。

例如：

- `/health` 不需要鉴权。
- `GET /todos` 可以公开。
- `POST /todos` 需要鉴权。

一种简单写法是创建两个 mux 或在注册时包装：

```go
mux.HandleFunc("GET /health", health)
mux.Handle("POST /todos", auth(http.HandlerFunc(createTodo)))
```

注意这里用的是 `Handle`，因为 `auth(...)` 返回的是 `http.Handler`。

---

## 三、CORS 是什么

CORS 全称 Cross-Origin Resource Sharing，跨源资源共享。

当前端页面来自：

```text
http://localhost:5173
```

后端 API 来自：

```text
http://localhost:8080
```

浏览器会认为它们不同源。前端调用后端时，浏览器会检查后端是否允许跨源访问。

CORS 是浏览器安全策略，不是 curl 的限制。用 curl 访问通常不会遇到 CORS 问题。

---

## 四、最小 CORS 中间件

学习阶段示例：

```go
func cors(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Access-Control-Allow-Origin", "http://localhost:5173")
		w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PATCH, DELETE, OPTIONS")
		w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Authorization, X-Request-ID")

		if r.Method == http.MethodOptions {
			w.WriteHeader(http.StatusNoContent)
			return
		}

		next.ServeHTTP(w, r)
	})
}
```

生产环境不要随意设置：

```text
Access-Control-Allow-Origin: *
```

尤其是涉及 Cookie 或用户数据时，要明确白名单。

---

## 五、OPTIONS 预检请求

浏览器在发送某些跨域请求前，会先发 `OPTIONS` 请求，问后端：

```text
我能不能用 POST？
我能不能带 Authorization？
我能不能带 Content-Type？
```

后端如果允许，就返回对应 CORS Header。

所以 CORS 中间件中经常会看到：

```go
if r.Method == http.MethodOptions {
	w.WriteHeader(http.StatusNoContent)
	return
}
```

---

## 六、中间件顺序建议

常见顺序：

```go
handler := chain(
	mux,
	recovery,
	requestID,
	logging,
	cors,
)
```

如果有鉴权，通常只包装需要保护的路由，而不是全局包住所有接口。

也可以根据项目风格做分组路由。标准库本身没有路由组，需要你自己组织。

---

## 七、本节检查点

请确认你能回答：

- `Authorization` Header 怎么读取？
- 鉴权失败为什么返回 `401`？
- CORS 是浏览器限制还是服务端限制？
- OPTIONS 预检请求是什么？
- 为什么生产环境要使用明确的 CORS 白名单？

