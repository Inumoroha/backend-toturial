# Go net/http 面试题与参考答案

这份文档用于系统复习 Go 标准库 `net/http` 相关面试问题。

它不只列“标准答案”，还会尽量说明面试官真正想考什么、回答时应该强调什么，以及常见误区。

适合复习目标：

```text
Go 后端工程师
Web 后端开发
API 服务开发
微服务 HTTP 接口开发
后端工程化岗位
```

---

## 一、复习建议

面试中关于 `net/http` 的问题通常分成几类：

```text
基础概念：HTTP、请求、响应、状态码。
标准库模型：Handler、HandlerFunc、ServeMux、Server。
工程实践：Middleware、统一错误、优雅关闭、超时。
客户端调用：http.Client、Transport、连接池。
Context：取消、超时、请求生命周期。
测试：httptest。
安全：CORS、Cookie、请求体限制、鉴权。
性能与排查：pprof、压测、连接复用。
源码与原理：Server 流程、Client 流程。
项目设计：Todo API、短链接、文件服务等。
```

复习时不要只背结论，要能结合项目讲：

```text
我在项目里怎么用？
为什么这样设计？
有什么坑？
如果上线还要补什么？
```

---

## 二、HTTP 基础类问题

### 1. HTTP 请求由哪些部分组成？

参考答案：

HTTP 请求通常由四部分组成：

```text
请求行
请求头 Header
空行
请求体 Body
```

例如：

```http
POST /todos?source=web HTTP/1.1
Host: localhost:8080
Content-Type: application/json
Authorization: Bearer token

{"title":"learn net/http"}
```

在 Go 中常见对应关系：

- `r.Method`：请求方法。
- `r.URL.Path`：路径。
- `r.URL.Query()`：Query 参数。
- `r.Header`：请求头。
- `r.Body`：请求体。

面试重点：

能把 HTTP 协议概念和 Go 的 `*http.Request` 对应起来。

---

### 2. HTTP 响应由哪些部分组成？

参考答案：

HTTP 响应通常由：

```text
状态行
响应头 Header
空行
响应体 Body
```

例如：

```http
HTTP/1.1 201 Created
Content-Type: application/json

{"id":1,"title":"learn"}
```

在 Go 中通过 `http.ResponseWriter` 写响应：

```go
w.Header().Set("Content-Type", "application/json")
w.WriteHeader(http.StatusCreated)
json.NewEncoder(w).Encode(data)
```

面试重点：

回答时要强调 Header、状态码、Body 的写入顺序。

---

### 3. GET 和 POST 有什么区别？

参考答案：

常见区别：

- `GET` 通常用于获取资源。
- `POST` 通常用于创建资源或提交动作。
- `GET` 参数常放在 URL Query 中。
- `POST` 数据常放在 Body 中。
- `GET` 语义上应该是安全、幂等的。
- `POST` 通常不是幂等的。

但要注意：

```text
GET 不是不能有 Body，只是不推荐也不通用。
POST 不是一定创建资源，也可以表达提交动作。
```

面试重点：

不要只说“GET 参数在 URL，POST 参数在 Body”。更好的回答要提到语义、安全性、幂等性和缓存。

---

### 4. 什么是幂等？

参考答案：

幂等表示同一个请求执行一次和执行多次，对资源状态的最终影响相同。

例子：

- `GET /todos/1`：幂等。
- `PUT /todos/1`：通常幂等，因为整体替换成同一个状态。
- `DELETE /todos/1`：通常幂等，删除一次和多次最终都是不存在。
- `POST /todos`：通常不幂等，多次调用会创建多个资源。

面试重点：

幂等说的是“资源状态的最终影响”，不是“响应内容完全一样”。

---

### 5. 常见 HTTP 状态码有哪些？

参考答案：

常见状态码：

- `200 OK`：请求成功。
- `201 Created`：资源创建成功。
- `204 No Content`：成功但没有响应体。
- `301 Moved Permanently`：永久重定向。
- `302 Found`：临时重定向。
- `400 Bad Request`：请求参数或格式错误。
- `401 Unauthorized`：未认证或认证失败。
- `403 Forbidden`：已认证但无权限。
- `404 Not Found`：资源不存在。
- `405 Method Not Allowed`：方法不允许。
- `409 Conflict`：资源冲突。
- `413 Payload Too Large`：请求体太大。
- `500 Internal Server Error`：服务端内部错误。
- `502 Bad Gateway`：网关收到上游错误响应。
- `503 Service Unavailable`：服务暂不可用。
- `504 Gateway Timeout`：网关或下游超时。

面试重点：

要能区分 `400`、`401`、`403`、`404`、`409`、`500`。

---

### 6. 401 和 403 有什么区别？

参考答案：

`401 Unauthorized` 表示认证失败：

```text
没有 Token
Token 无效
Token 过期
```

`403 Forbidden` 表示认证成功，但没有权限：

```text
普通用户访问管理员接口
用户访问不属于自己的资源
```

面试重点：

很多人会混用。清楚地区分认证和授权，是后端基本功。

---

### 7. 400 和 404 有什么区别？

参考答案：

`400 Bad Request` 表示客户端请求格式或参数错误。

例如：

```text
GET /todos/abc
```

如果 `id` 必须是数字，`abc` 是参数格式错误，返回 `400`。

`404 Not Found` 表示请求格式没问题，但资源不存在。

例如：

```text
GET /todos/999
```

`999` 是合法 ID，但数据库里没有这个 Todo，返回 `404`。

---

### 8. Header 和 Body 分别适合放什么？

参考答案：

Header 适合放元信息：

- `Content-Type`
- `Authorization`
- `Cookie`
- `User-Agent`
- `X-Request-ID`

Body 适合放业务数据：

- JSON 请求体。
- 表单内容。
- 文件内容。
- HTML 或 JSON 响应。

不要把大块业务数据塞进 Header。Header 有大小限制，也不适合表达复杂数据结构。

---

### 9. Query 参数和 Path 参数怎么选择？

参考答案：

Path 参数适合表示资源身份：

```text
GET /users/123
GET /todos/1
```

Query 参数适合表示过滤、分页、排序：

```text
GET /todos?done=false&page=1&page_size=20
GET /search?q=golang
```

经验规则：

```text
资源是谁，放 Path。
怎么筛选，放 Query。
创建和更新的大块数据，放 Body。
```

---

## 三、net/http 核心模型

### 10. `http.Handler` 是什么？

参考答案：

`http.Handler` 是 `net/http` 的核心接口：

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

只要一个类型实现了 `ServeHTTP(http.ResponseWriter, *http.Request)` 方法，它就是一个 HTTP Handler。

请求最终都会被交给某个 Handler 的 `ServeHTTP` 方法处理。

面试重点：

要强调 `Handler` 是 Go HTTP 服务的中心抽象，路由器、中间件、业务处理函数都围绕它组合。

---

### 11. `http.HandlerFunc` 是什么？

参考答案：

`HandlerFunc` 是一个函数类型，用来把普通函数适配成 `http.Handler`。

源码思想：

```go
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

所以普通函数：

```go
func hello(w http.ResponseWriter, r *http.Request) {}
```

可以通过：

```go
http.HandlerFunc(hello)
```

变成一个 Handler。

---

### 12. `Handler` 和 `HandlerFunc` 的关系是什么？

参考答案：

`Handler` 是接口，`HandlerFunc` 是函数适配器。

关系：

```text
普通函数
-> 转成 HandlerFunc
-> HandlerFunc 实现 ServeHTTP
-> 满足 Handler 接口
```

这让函数、结构体、路由器、中间件都能用统一的 `http.Handler` 模型组合。

---

### 13. `http.ServeMux` 是什么？

参考答案：

`ServeMux` 是标准库提供的请求多路复用器，也就是路由器。

它负责：

```text
接收请求
-> 根据 Method 和 Path 匹配路由
-> 找到对应 Handler
-> 调用 Handler.ServeHTTP
```

`ServeMux` 本身也实现了 `http.Handler`，所以可以传给 `http.Server` 或被中间件包装。

---

### 14. `http.HandleFunc` 和 `mux.HandleFunc` 有什么区别？

参考答案：

`http.HandleFunc` 注册到全局默认路由器 `http.DefaultServeMux`。

```go
http.HandleFunc("/hello", hello)
http.ListenAndServe(":8080", nil)
```

`mux.HandleFunc` 注册到自己创建的 `ServeMux`：

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /hello", hello)
http.ListenAndServe(":8080", mux)
```

项目中更推荐自定义 mux，因为：

- 依赖更明确。
- 更容易测试。
- 避免全局路由污染。
- 可以为不同服务创建不同 mux。

---

### 15. `ListenAndServe(":8080", nil)` 中 nil 表示什么？

参考答案：

第二个参数是 Handler。

如果传 `nil`，标准库会使用默认的：

```go
http.DefaultServeMux
```

也就是说：

```go
http.ListenAndServe(":8080", nil)
```

会把请求交给全局默认路由器处理。

项目中不建议长期依赖默认 mux。

---

### 16. Go 1.22 的 ServeMux 有什么变化？

参考答案：

Go 1.22 的 `ServeMux` 支持更现代的路由模式：

```go
mux.HandleFunc("GET /todos", listTodos)
mux.HandleFunc("POST /todos", createTodo)
mux.HandleFunc("GET /todos/{id}", getTodo)
```

可以读取路径参数：

```go
id := r.PathValue("id")
```

这让标准库更适合直接写 RESTful API。

---

### 17. `http.ResponseWriter` 是什么？

参考答案：

`ResponseWriter` 用来写 HTTP 响应。

常见操作：

```go
w.Header().Set("Content-Type", "application/json")
w.WriteHeader(http.StatusCreated)
w.Write([]byte(`{"id":1}`))
```

注意：

- Header 要在 `WriteHeader` 或 `Write` 前设置。
- 如果直接 `Write`，默认状态码是 `200 OK`。
- `WriteHeader` 只有第一次有效。

---

### 18. `*http.Request` 里常用字段有哪些？

参考答案：

常用字段和方法：

- `r.Method`：请求方法。
- `r.URL.Path`：路径。
- `r.URL.Query()`：Query 参数。
- `r.Header.Get(...)`：读取 Header。
- `r.Body`：请求体。
- `r.Context()`：请求上下文。
- `r.RemoteAddr`：直接连接方地址。
- `r.UserAgent()`：User-Agent。
- `r.Cookie(...)`：读取 Cookie。
- `r.PathValue(...)`：读取路径参数。

---

## 四、请求与响应处理

### 19. 如何读取 JSON 请求体？

参考答案：

```go
type createTodoRequest struct {
	Title string `json:"title"`
}

func createTodo(w http.ResponseWriter, r *http.Request) {
	var req createTodoRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "invalid json", http.StatusBadRequest)
		return
	}
}
```

真实项目建议先限制请求体大小：

```go
r.Body = http.MaxBytesReader(w, r.Body, 1<<20)
```

---

### 20. 为什么要限制请求体大小？

参考答案：

如果不限制请求体，客户端可以发送很大的 Body，占用：

- 内存。
- CPU。
- 网络带宽。
- goroutine。
- 临时文件空间。

限制方式：

```go
r.Body = http.MaxBytesReader(w, r.Body, 1<<20)
```

这属于 Web 服务的基础安全措施。

---

### 21. 如何返回 JSON 响应？

参考答案：

```go
func writeJSON(w http.ResponseWriter, status int, data any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	_ = json.NewEncoder(w).Encode(data)
}
```

使用：

```go
writeJSON(w, http.StatusCreated, todo)
```

注意顺序：

```text
先 Header
再状态码
最后 Body
```

---

### 22. 为什么 `WriteHeader` 要在 `Write` 之前调用？

参考答案：

如果先调用 `Write`，Go 会自动发送响应头，并默认状态码为 `200 OK`。

之后再调用：

```go
w.WriteHeader(http.StatusBadRequest)
```

已经无法改变状态码。

所以错误路径要在写响应体前判断并返回。

---

### 23. `WriteHeader` 调用多次会怎样？

参考答案：

只有第一次有效。

例如：

```go
w.WriteHeader(http.StatusCreated)
w.WriteHeader(http.StatusInternalServerError)
```

客户端看到的是 `201 Created`。

后续调用通常会被忽略，并可能在日志中出现 warning。

---

### 24. 如何设计统一错误响应？

参考答案：

可以统一成：

```json
{
  "error": {
    "code": "invalid_request",
    "message": "title is required"
  }
}
```

Go 封装：

```go
func writeError(w http.ResponseWriter, status int, code, message string) {
	writeJSON(w, status, map[string]any{
		"error": map[string]string{
			"code":    code,
			"message": message,
		},
	})
}
```

好处：

- 客户端处理统一。
- 错误码稳定。
- 不暴露内部错误。

---

### 25. 为什么不要把内部错误直接返回给客户端？

参考答案：

内部错误可能包含：

- SQL 语句。
- 表名。
- 文件路径。
- 堆栈信息。
- 内部服务地址。
- 敏感业务信息。

不推荐：

```json
{"error":"pq: duplicate key value violates unique constraint users_email_key"}
```

推荐：

```json
{"error":{"code":"email_conflict","message":"email already exists"}}
```

内部错误写日志，对外返回稳定错误。

---

### 26. 如何读取 Query 参数？

参考答案：

```go
q := r.URL.Query().Get("q")
page := r.URL.Query().Get("page")
```

注意：

- Query 参数都是字符串。
- 需要转换类型。
- 需要校验默认值和非法值。

例如：

```go
page, err := strconv.Atoi(r.URL.Query().Get("page"))
if err != nil || page <= 0 {
	writeError(w, http.StatusBadRequest, "invalid_page", "invalid page")
	return
}
```

---

### 27. 如何读取路径参数？

参考答案：

Go 1.22 以后：

```go
mux.HandleFunc("GET /todos/{id}", getTodo)
```

读取：

```go
idText := r.PathValue("id")
```

然后转换和校验：

```go
id, err := strconv.Atoi(idText)
if err != nil || id <= 0 {
	writeError(w, http.StatusBadRequest, "invalid_id", "invalid id")
	return
}
```

---

### 28. 如何读取和设置 Header？

参考答案：

读取：

```go
auth := r.Header.Get("Authorization")
ua := r.Header.Get("User-Agent")
```

设置响应 Header：

```go
w.Header().Set("Content-Type", "application/json")
w.Header().Set("X-Request-ID", requestID)
```

Header 要在写响应体前设置。

---

### 29. 如何设置 Cookie？

参考答案：

```go
http.SetCookie(w, &http.Cookie{
	Name:     "session_id",
	Value:    "abc123",
	Path:     "/",
	HttpOnly: true,
	Secure:   true,
	SameSite: http.SameSiteLaxMode,
})
```

常见安全属性：

- `HttpOnly`：禁止 JavaScript 读取。
- `Secure`：只在 HTTPS 发送。
- `SameSite`：缓解 CSRF。
- `MaxAge`：有效期。

---

### 30. 如何读取 Cookie？

参考答案：

```go
cookie, err := r.Cookie("session_id")
if err != nil {
	writeError(w, http.StatusUnauthorized, "unauthorized", "missing session")
	return
}

sessionID := cookie.Value
```

如果 Cookie 不存在，会返回错误。

---

## 五、Middleware

### 31. Middleware 的本质是什么？

参考答案：

Middleware 的本质是：

```go
func(http.Handler) http.Handler
```

它接收一个 Handler，返回一个新的 Handler。

典型写法：

```go
func logging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Println(r.Method, r.URL.Path)
		next.ServeHTTP(w, r)
	})
}
```

---

### 32. Middleware 适合处理什么？

参考答案：

适合处理横切逻辑：

- 请求日志。
- Request ID。
- Panic Recovery。
- 鉴权。
- CORS。
- 限流。
- 安全 Header。
- 请求耗时统计。

不适合放：

- 具体业务逻辑。
- 某个接口独有的复杂校验。
- 数据库 CRUD 细节。

---

### 33. 多个 Middleware 的执行顺序如何判断？

参考答案：

如果：

```go
handler := recovery(logging(auth(mux)))
```

请求进入顺序：

```text
recovery -> logging -> auth -> mux
```

响应返回时按调用栈反向返回。

如果使用 Chain 函数，要看它是从前往后包还是从后往前包。

---

### 34. 鉴权中间件什么时候不调用 next？

参考答案：

鉴权失败时不应该继续执行业务 Handler。

```go
func auth(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.Header.Get("Authorization") != "Bearer dev-token" {
			writeError(w, http.StatusUnauthorized, "unauthorized", "invalid token")
			return
		}
		next.ServeHTTP(w, r)
	})
}
```

如果不 `return`，可能会继续执行业务逻辑，造成安全问题。

---

### 35. Recovery Middleware 有什么作用？

参考答案：

Recovery Middleware 用来捕获 Handler 中的 panic，避免单个请求异常导致响应不可控。

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

注意：

- 它不能替代正常错误处理。
- 不要把 panic 内容直接返回给客户端。
- 如果响应已经写出，再 panic，状态码可能无法改成 500。

---

### 36. 如何在 Middleware 中记录响应状态码？

参考答案：

需要包装 `http.ResponseWriter`：

```go
type statusRecorder struct {
	http.ResponseWriter
	status int
}

func (r *statusRecorder) WriteHeader(status int) {
	r.status = status
	r.ResponseWriter.WriteHeader(status)
}
```

使用：

```go
rec := &statusRecorder{ResponseWriter: w, status: http.StatusOK}
next.ServeHTTP(rec, r)
log.Println(rec.status)
```

默认 `status` 设为 `200`，因为 Handler 直接 `Write` 时默认返回 `200 OK`。

---

### 37. Request ID 中间件怎么设计？

参考答案：

设计思路：

- 如果请求带 `X-Request-ID`，透传。
- 如果没有，服务端生成。
- 写入响应 Header。
- 放入 context，供日志使用。

示例：

```go
func requestID(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		id := r.Header.Get("X-Request-ID")
		if id == "" {
			id = generateID()
		}
		w.Header().Set("X-Request-ID", id)
		ctx := context.WithValue(r.Context(), requestIDKey, id)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}
```

---

## 六、http.Server 与优雅关闭

### 38. 为什么生产环境不建议只用 `http.ListenAndServe` 简写？

参考答案：

简写：

```go
http.ListenAndServe(":8080", handler)
```

适合学习，但生产环境通常需要显式配置：

- `ReadHeaderTimeout`
- `ReadTimeout`
- `WriteTimeout`
- `IdleTimeout`
- `MaxHeaderBytes`
- 优雅关闭

推荐：

```go
srv := &http.Server{
	Addr:              ":8080",
	Handler:           handler,
	ReadHeaderTimeout: 5 * time.Second,
	ReadTimeout:       10 * time.Second,
	WriteTimeout:      10 * time.Second,
	IdleTimeout:       60 * time.Second,
}
```

---

### 39. `ReadHeaderTimeout`、`ReadTimeout`、`WriteTimeout`、`IdleTimeout` 分别是什么？

参考答案：

- `ReadHeaderTimeout`：读取请求头的最大时间，防慢速 Header 攻击。
- `ReadTimeout`：读取整个请求的最大时间，包括 Header 和 Body。
- `WriteTimeout`：写响应的最大时间。
- `IdleTimeout`：keep-alive 空闲连接保留时间。

面试重点：

能说清楚它们作用在 HTTP 请求生命周期的不同阶段。

---

### 40. 什么是优雅关闭？

参考答案：

优雅关闭指服务退出时：

```text
停止接收新请求
等待正在处理的请求完成
关闭空闲连接
在超时时间内退出
```

Go 中使用：

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
srv.Shutdown(ctx)
```

---

### 41. `Shutdown` 和 `Close` 有什么区别？

参考答案：

`Shutdown`：

- 优雅关闭。
- 不再接收新请求。
- 等待活跃请求完成。
- 推荐用于正常退出。

`Close`：

- 立即关闭。
- 活跃连接也会被断开。
- 更像强制停止。

---

### 42. 为什么 `ListenAndServe` 要放到 goroutine 中？

参考答案：

`ListenAndServe` 会阻塞当前 goroutine。

如果主 goroutine 还要监听退出信号，就需要：

```go
go func() {
	if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
		log.Fatal(err)
	}
}()
```

主 goroutine 再执行：

```go
<-ctx.Done()
srv.Shutdown(...)
```

---

### 43. `http.ErrServerClosed` 表示什么？

参考答案：

当调用 `Shutdown` 或 `Close` 正常关闭 Server 时，`ListenAndServe` 会返回：

```go
http.ErrServerClosed
```

这不是异常崩溃，不应该当成 fatal 错误。

常见判断：

```go
if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
	log.Fatal(err)
}
```

---

### 44. 如何处理系统退出信号？

参考答案：

可以用：

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer stop()

<-ctx.Done()
```

收到 `Ctrl+C` 或 `SIGTERM` 后，调用 `srv.Shutdown`。

---

## 七、Context

### 45. Handler 中为什么要用 `r.Context()`？

参考答案：

`r.Context()` 表示当前请求的生命周期。

它会在这些情况下取消：

- 客户端断开连接。
- 请求被服务端取消。
- Server shutdown。
- 上游 context 取消。

下游调用应该使用 `r.Context()`，这样请求取消后，数据库查询、HTTP 调用等可以及时停止。

---

### 46. 为什么不要在 Handler 中随便使用 `context.Background()`？

参考答案：

如果在 Handler 中使用：

```go
context.Background()
```

下游任务就和当前请求生命周期断开了。

客户端断开或请求超时后，下游任务可能还在继续执行，浪费资源。

推荐：

```go
service.Do(r.Context())
```

---

### 47. `context.Canceled` 和 `context.DeadlineExceeded` 有什么区别？

参考答案：

`context.Canceled`：

```text
context 被主动取消。
例如客户端断开、调用 cancel。
```

`context.DeadlineExceeded`：

```text
context 超过截止时间。
例如 WithTimeout 超时。
```

---

### 48. 如何给下游调用设置超时？

参考答案：

```go
ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
defer cancel()

result, err := client.Call(ctx)
```

父 context 使用 `r.Context()`，这样同时继承：

- 客户端取消。
- 服务关闭。
- 当前下游调用超时。

---

### 49. `context.Value` 适合存什么？

参考答案：

适合存请求范围内的少量元信息：

- Request ID。
- Trace ID。
- 当前用户 ID。
- 租户 ID。

不适合存：

- 大对象。
- 配置。
- 数据库连接池。
- 普通业务参数。

业务参数应该显式传参。

---

## 八、HTTP Client

### 50. `http.Client` 应该每次请求都新建吗？

参考答案：

不应该。

`http.Client` 应该复用，因为它底层通过 `Transport` 管理连接池。

每次新建 Client 可能导致：

- 连接无法复用。
- 频繁 TCP/TLS 握手。
- 资源浪费。

推荐：

```go
client := &http.Client{
	Timeout: 5 * time.Second,
}
```

作为结构体字段长期复用。

---

### 51. `http.Client.Timeout` 控制什么？

参考答案：

`Client.Timeout` 控制整个请求生命周期的大致总超时，包括：

- 建立连接。
- 重定向。
- 读取响应头。
- 读取响应体。

真实项目中还可以结合 context 做单次调用超时。

---

### 52. 为什么 HTTP Client 请求要使用 `NewRequestWithContext`？

参考答案：

这样下游 HTTP 调用能响应取消和超时。

```go
req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
resp, err := client.Do(req)
```

如果上游请求取消，下游 HTTP 请求也可以停止。

---

### 53. 为什么必须关闭 `resp.Body`？

参考答案：

拿到响应后必须：

```go
defer resp.Body.Close()
```

原因：

- 释放连接资源。
- 让 Transport 有机会复用连接。
- 避免资源泄漏。

只要 `err == nil` 且 `resp != nil`，即使状态码是 404 或 500，也要关闭 Body。

---

### 54. `client.Do` 遇到 404 会返回 error 吗？

参考答案：

通常不会。

`client.Do` 的 error 主要表示：

- 网络错误。
- DNS 错误。
- 连接失败。
- 超时。
- context 取消。
- 请求构造问题。

HTTP 状态码如 `404`、`500` 仍然是合法 HTTP 响应，需要自己检查：

```go
if resp.StatusCode < 200 || resp.StatusCode >= 300 {
	return fmt.Errorf("unexpected status: %d", resp.StatusCode)
}
```

---

### 55. `http.Transport` 是什么？

参考答案：

`Transport` 是 `http.Client` 底层执行请求和管理连接的组件。

它负责：

- 建立 TCP 连接。
- TLS 握手。
- 连接池。
- keep-alive。
- 代理。
- HTTP/2 支持。

默认 Client 也有 Transport，但高并发或特殊场景可以自定义。

---

### 56. `RoundTripper` 是什么？

参考答案：

`RoundTripper` 是 HTTP Client 底层接口：

```go
type RoundTripper interface {
	RoundTrip(*Request) (*Response, error)
}
```

`http.Transport` 实现了它。

它也可以用于测试或自定义客户端行为。

---

## 九、测试

### 57. 如何测试一个 Handler？

参考答案：

使用 `httptest.NewRequest` 和 `httptest.NewRecorder`。

```go
req := httptest.NewRequest(http.MethodGet, "/health", nil)
rec := httptest.NewRecorder()

handler.ServeHTTP(rec, req)

resp := rec.Result()
defer resp.Body.Close()
```

然后断言：

- 状态码。
- Header。
- Body。
- JSON 字段。

---

### 58. `httptest.NewRecorder` 是什么？

参考答案：

`httptest.NewRecorder()` 返回一个 `ResponseRecorder`，它实现了 `http.ResponseWriter`。

它可以记录 Handler 写出的：

- 状态码。
- Header。
- Body。

适合 Handler 单元测试。

---

### 59. `httptest.NewServer` 适合什么场景？

参考答案：

`httptest.NewServer` 会启动一个真实测试 HTTP 服务。

适合：

- 测试 HTTP Client。
- 测完整路由和中间件链。
- 模拟第三方 API。
- 做轻量集成测试。

示例：

```go
server := httptest.NewServer(handler)
defer server.Close()

resp, err := http.Get(server.URL + "/health")
```

---

### 60. 如何测试 Middleware？

参考答案：

中间件本质也是 Handler 包装，所以构造一个假的 next Handler。

```go
nextCalled := false
next := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	nextCalled = true
})

auth(next).ServeHTTP(rec, req)
```

断言：

- 鉴权失败时 next 不被调用。
- 鉴权成功时 next 被调用。
- 状态码正确。

---

## 十、文件上传下载

### 61. 如何处理文件上传？

参考答案：

常见使用 multipart：

```go
r.Body = http.MaxBytesReader(w, r.Body, maxSize)
if err := r.ParseMultipartForm(maxSize); err != nil {
	writeError(w, http.StatusBadRequest, "invalid_form", "invalid form")
	return
}

file, header, err := r.FormFile("file")
if err != nil {
	writeError(w, http.StatusBadRequest, "missing_file", "missing file")
	return
}
defer file.Close()
```

还要校验：

- 文件大小。
- 文件名。
- 后缀。
- Content-Type。

---

### 62. 为什么不能直接使用上传文件名？

参考答案：

用户上传的文件名不可信，可能包含路径穿越：

```text
../../secret.txt
```

如果直接拼接路径，可能写到非预期目录。

至少使用：

```go
filepath.Base(header.Filename)
```

更推荐服务端生成文件名。

---

### 63. `http.ServeFile` 和 `http.FileServer` 有什么区别？

参考答案：

`http.ServeFile`：

```text
返回单个指定文件。
适合下载接口。
可以先做权限校验再返回。
```

`http.FileServer`：

```text
暴露整个目录。
适合公开静态资源。
不适合直接暴露私有文件。
```

---

### 64. 如何防止文件下载路径穿越？

参考答案：

不要直接拼接用户输入：

```go
path := "./uploads/" + name
```

至少：

```go
name := filepath.Base(r.PathValue("name"))
path := filepath.Join("./uploads", name)
```

更严格做法是通过数据库记录查找文件，而不是让用户直接控制文件路径。

---

## 十一、安全

### 65. CORS 是什么？

参考答案：

CORS 是浏览器的跨源资源共享机制。

当前端页面和后端 API 不同源时，浏览器会检查后端是否允许跨源访问。

后端常设置：

```text
Access-Control-Allow-Origin
Access-Control-Allow-Methods
Access-Control-Allow-Headers
```

注意：

```text
CORS 是浏览器限制，不是服务端鉴权。
curl 不受 CORS 限制。
```

---

### 66. CORS 能替代鉴权吗？

参考答案：

不能。

CORS 只是浏览器安全策略。攻击者可以用：

- curl。
- Postman。
- 自己写程序。

直接请求后端。

敏感接口必须在服务端做认证和授权。

---

### 67. Cookie 的 `HttpOnly`、`Secure`、`SameSite` 分别有什么用？

参考答案：

- `HttpOnly`：禁止 JavaScript 读取 Cookie，降低 XSS 后 Cookie 被窃取风险。
- `Secure`：只在 HTTPS 下发送 Cookie。
- `SameSite`：控制跨站请求是否携带 Cookie，缓解 CSRF。

生产 Session Cookie 通常应该设置这些属性。

---

### 68. 什么是 SSRF？短链接服务为什么要注意？

参考答案：

SSRF 是服务端请求伪造。

短链接服务如果允许用户提交任意 URL，可能被用来跳转或探测：

```text
http://127.0.0.1:8080/admin
http://169.254.169.254/
http://internal-service/
```

防护思路：

- 只允许 `http` 和 `https`。
- 禁止内网 IP。
- 禁止 localhost。
- 做 DNS 解析和 IP 校验。
- 对跳转目标做安全策略。

---

### 69. 为什么日志里不能打印完整 Token？

参考答案：

Token 属于敏感凭证。

日志可能被：

- 多人查看。
- 上传到日志平台。
- 长期保存。
- 被攻击者获取。

如果必须记录，只记录是否存在、用户 ID，或脱敏后的少量片段。

---

## 十二、性能与观测

### 70. 如何排查一个接口变慢？

参考答案：

可以按顺序排查：

```text
看请求日志，确认哪个 path 慢。
看 status，确认是否错误变多。
看 request_id，关联业务日志。
看下游调用耗时。
看数据库慢查询。
用 pprof 看 CPU/内存/goroutine。
用压测工具复现。
```

不要凭感觉优化。

---

### 71. pprof 可以分析什么？

参考答案：

pprof 可以分析：

- CPU。
- heap 内存。
- goroutine。
- block。
- mutex。
- trace。

HTTP 服务中常通过：

```go
import _ "net/http/pprof"
```

暴露调试接口。

生产环境不要把 pprof 暴露到公网。

---

### 72. 压测时为什么不能只看平均耗时？

参考答案：

平均耗时会掩盖长尾问题。

应该关注：

- P50。
- P95。
- P99。
- 最大耗时。
- 错误率。

例如平均 20ms，但 P99 是 2s，用户体验仍然很差。

---

### 73. Go HTTP 服务中共享 map 为什么要加锁？

参考答案：

HTTP 请求会并发处理。

普通 Go map 不能并发读写，否则可能：

- data race。
- panic。
- 数据错误。

需要使用：

```go
sync.Mutex
sync.RWMutex
sync.Map
```

或者把数据放到数据库中。

可以用：

```bash
go test -race ./...
```

检查 data race。

---

### 74. 如何做基础限流？

参考答案：

常见限流维度：

- IP。
- 用户 ID。
- Token。
- 接口路径。

常见算法：

- 固定窗口。
- 滑动窗口。
- 令牌桶。
- 漏桶。

Go 中可以用中间件实现，或者借助网关、Redis、Nginx。

面试时要强调：单机内存限流无法覆盖多实例，需要 Redis 或网关层统一限流。

---

## 十三、源码与原理

### 75. `net/http` 服务端大致处理流程是什么？

参考答案：

简化流程：

```text
ListenAndServe
-> net.Listen
-> Accept connection
-> 为连接启动 goroutine
-> 读取 HTTP 请求
-> 找到 Handler
-> 调用 ServeHTTP
-> 写响应
```

核心是：请求最终会进入 Handler 的 `ServeHTTP`。

---

### 76. 为什么 ServeMux 也能被 Middleware 包装？

参考答案：

因为 `ServeMux` 实现了 `http.Handler`。

中间件接收的是：

```go
http.Handler
```

所以无论是：

- `ServeMux`
- `HandlerFunc`
- 自定义 Handler

都可以被统一包装。

---

### 77. `http.Client` 的大致流程是什么？

参考答案：

简化流程：

```text
构造 Request
-> Client.Do
-> Transport.RoundTrip
-> 建立或复用连接
-> 发送请求
-> 接收响应
-> 返回 Response
-> 调用方关闭 Body
```

Client 负责高层流程，Transport 负责底层连接和请求往返。

---

### 78. 为什么 `net/http` 的设计适合组合？

参考答案：

因为它围绕一个很小的接口：

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

路由器是 Handler，中间件返回 Handler，业务函数可以适配为 Handler。

小接口带来强组合性。

---

## 十四、项目设计类问题

### 79. 如何设计一个 Todo API？

参考答案：

接口：

```text
GET    /todos
POST   /todos
GET    /todos/{id}
PATCH  /todos/{id}
DELETE /todos/{id}
```

分层：

```text
Handler：HTTP 输入输出。
Service：业务规则。
Store：数据访问。
Middleware：日志、鉴权、Recovery。
```

工程能力：

- JSON 统一响应。
- 错误统一格式。
- `http.Server` 超时。
- 优雅关闭。
- `httptest` 测试。

---

### 80. 如何设计短链接服务？

参考答案：

核心流程：

```text
提交长链接
-> 生成短码
-> 保存映射
-> 访问短码
-> 302 跳转
-> 更新访问次数
```

接口：

```text
POST /links
GET  /{code}
GET  /links/{code}
GET  /links
```

关键点：

- 短码唯一性。
- URL 校验。
- 访问计数并发安全。
- 跳转返回 `302` 和 `Location`。
- 数据库存储时 `code` 要唯一索引。

---

### 81. 短码如何保证唯一？

参考答案：

不能只靠应用层“先查再插”。

并发下两个请求可能同时认为 code 不存在。

正确做法：

```text
数据库 UNIQUE(code) 兜底。
应用层捕获唯一冲突后重试生成。
```

内存版本也要在锁内检查和插入。

---

### 82. 访问次数如何并发安全更新？

参考答案：

内存版本：

```go
mu.Lock()
link.VisitCount++
mu.Unlock()
```

数据库版本：

```sql
UPDATE links
SET visit_count = visit_count + 1
WHERE code = $1;
```

不要先查出计数，在应用里加 1，再写回。并发下容易丢失更新。

---

### 83. 如何设计文件上传服务？

参考答案：

接口：

```text
POST /upload
GET /files/{name}
GET /static/*
```

关键点：

- 限制 Body 大小。
- 校验 multipart。
- 不信任文件名。
- 校验后缀和 Content-Type。
- 服务端生成文件名。
- 私有文件下载前做权限校验。
- 静态目录只放公开文件。

---

### 84. 如何设计统一错误码？

参考答案：

建议错误响应稳定：

```json
{
  "error": {
    "code": "todo_not_found",
    "message": "todo not found"
  }
}
```

错误码给程序判断，message 给人看。

设计原则：

- 不暴露内部错误。
- 不频繁改 code。
- HTTP 状态码和业务错误保持一致。
- 日志中保留内部错误细节。

---

## 十五、高频陷阱题

### 85. Handler 写了错误响应后忘记 return 会怎样？

参考答案：

可能继续执行后面的成功逻辑，导致：

- 重复写响应。
- 状态码异常。
- 业务逻辑误执行。
- 安全漏洞。

错误写法：

```go
if err != nil {
	writeError(w, 400, "bad_request", "bad request")
}
// 这里还会继续执行
```

正确：

```go
if err != nil {
	writeError(w, 400, "bad_request", "bad request")
	return
}
```

---

### 86. Header 在 WriteHeader 后设置还生效吗？

参考答案：

通常不会。

因为 `WriteHeader` 会发送响应头。之后再设置 Header 太晚。

正确顺序：

```go
w.Header().Set("Content-Type", "application/json")
w.WriteHeader(http.StatusCreated)
w.Write(body)
```

---

### 87. `r.Body` 可以读多次吗？

参考答案：

通常不能。

`r.Body` 是流，读完一次就消耗掉了。

如果确实要读多次，需要把内容读出来后重新构造：

```go
data, _ := io.ReadAll(r.Body)
r.Body = io.NopCloser(bytes.NewReader(data))
```

但不建议随意这么做，大 Body 会占内存。

---

### 88. `http.Client` 的零值能用吗？

参考答案：

能用。

例如：

```go
http.DefaultClient
```

但默认 Client 没有全局 Timeout。生产调用第三方服务时，建议显式设置超时。

---

### 89. `http.DefaultServeMux` 有什么风险？

参考答案：

它是全局变量。

风险：

- 不同包可能注册路由到同一个默认 mux。
- 测试之间可能互相影响。
- 项目依赖不够明确。

项目中推荐使用：

```go
mux := http.NewServeMux()
```

---

### 90. Recovery 能捕获所有错误吗？

参考答案：

不能。

Recovery 只能捕获当前 goroutine 中的 panic。

如果 Handler 里启动新的 goroutine：

```go
go func() {
	panic("boom")
}()
```

外层 recovery 捕获不到。新 goroutine 内部需要自己 recover。

---

### 91. `RemoteAddr` 一定是真实客户端 IP 吗？

参考答案：

不一定。

如果服务在 Nginx、负载均衡、网关后面，`r.RemoteAddr` 可能是代理地址。

真实 IP 可能在：

```text
X-Forwarded-For
X-Real-IP
```

但这些 Header 可以伪造，只有来自可信代理时才应该信任。

---

### 92. 用 `http.Error` 返回 JSON API 合适吗？

参考答案：

`http.Error` 返回的是纯文本，适合简单接口或早期学习。

JSON API 更推荐统一封装：

```go
writeError(w, status, code, message)
```

这样客户端处理更一致。

---

### 93. 为什么 `ResponseWriter` 包装要注意额外接口？

参考答案：

`ResponseWriter` 可能还实现了额外接口，比如：

- `http.Flusher`
- `http.Hijacker`
- `http.Pusher`
- `io.ReaderFrom`

简单包装可能丢失这些能力，影响 WebSocket、流式响应等高级场景。

普通 JSON API 中问题不大，但写通用中间件要注意。

---

### 94. `MaxBytesReader` 和 `io.LimitReader` 有什么区别？

参考答案：

`io.LimitReader` 只是限制读取量，超过部分不会自动给客户端明确 HTTP 错误。

`http.MaxBytesReader` 专门用于 HTTP 请求体限制，会在超过限制时返回错误，并能配合 HTTP Server 处理连接。

在 Handler 中限制请求体大小，更推荐 `http.MaxBytesReader`。

---

### 95. `json.Decoder` 和 `json.Unmarshal` 读取请求体有什么区别？

参考答案：

`json.NewDecoder(r.Body).Decode(&dst)` 可以直接从流读取。

`json.Unmarshal` 需要先：

```go
data, _ := io.ReadAll(r.Body)
json.Unmarshal(data, &dst)
```

对于 HTTP Body，Decoder 更直接。

但如果需要严格检查多余内容、限制大小、记录原始 Body，可能会结合 ReadAll 使用。要注意内存风险。

---

## 十六、开放题参考回答

### 96. 你会如何从零搭建一个 Go HTTP 服务？

参考答案：

我会按这个顺序：

```text
初始化 go module。
创建 cmd/server/main.go。
创建 http.NewServeMux 注册路由。
封装 writeJSON/writeError。
拆 Handler、Service、Store。
增加 middleware：request id、logging、recovery。
显式配置 http.Server 超时。
处理 signal 和 graceful shutdown。
补 httptest 测试。
写 README 和 curl 示例。
```

如果是生产服务，还会补：

- 配置管理。
- 结构化日志。
- 指标。
- pprof 内网暴露。
- Dockerfile。
- CI 测试。

---

### 97. 如果线上接口偶发 500，你怎么排查？

参考答案：

排查顺序：

```text
看请求日志，定位 path、status、request_id。
用 request_id 查错误日志。
确认是否 panic，被 recovery 捕获。
查看最近发布变更。
查看下游依赖：数据库、缓存、第三方 API。
查看错误率是否集中在某个接口或参数。
必要时复现请求并补测试。
```

如果是并发问题，会加 race 检测、检查共享变量和锁。

---

### 98. 如果接口 P99 很高但平均耗时正常，你怎么处理？

参考答案：

说明存在长尾请求。

排查：

- 按 duration 筛选慢请求日志。
- 看慢请求集中在哪个 path。
- 看是否某些参数导致慢。
- 查下游数据库慢查询。
- 查 HTTP Client 下游耗时。
- 用 pprof 看 CPU 和 goroutine。
- 检查锁竞争和连接池耗尽。

优化：

- 给下游调用设置超时。
- 优化慢 SQL。
- 加缓存。
- 减少锁粒度。
- 对大响应分页或流式返回。

---

### 99. 标准库和 Gin/框架怎么选择？

参考答案：

标准库优点：

- 依赖少。
- 模型清晰。
- 和 Go 原生生态兼容。
- 适合学习和中小服务。

框架优点：

- 路由组更方便。
- 参数绑定更方便。
- 中间件生态更丰富。
- 开发效率更高。

我的选择：

```text
如果项目简单、团队熟悉标准库，用 net/http 足够。
如果业务接口很多、需要成熟生态和开发效率，可以用 Gin/Echo/Chi。
但无论用什么框架，都要理解底层 Handler、Request、ResponseWriter、Context。
```

---

### 100. 你认为一个生产级 HTTP 服务至少应该具备什么？

参考答案：

至少应该具备：

- 明确路由。
- 统一 JSON 响应。
- 统一错误响应。
- 请求体大小限制。
- Server 超时配置。
- 优雅关闭。
- 结构化日志。
- Request ID。
- Panic Recovery。
- 鉴权和权限控制。
- 基础安全 Header 或 CORS 白名单。
- Handler/Middleware 测试。
- 健康检查接口。
- 指标和 pprof。
- README、部署说明和配置说明。

如果有外部调用，还要：

- 复用 HTTP Client。
- 设置超时。
- 正确关闭响应体。
- 处理非 2xx 状态码。

---

## 十七、快速复习清单

面试前建议快速过一遍：

```text
Handler / HandlerFunc / ServeMux
ResponseWriter 写响应顺序
Request 读取 Method、Path、Query、Header、Body
JSON Decode / Encode
writeJSON / writeError
Middleware 模型
Recovery
Request ID
http.Server 超时
Graceful Shutdown
r.Context()
http.Client 复用
resp.Body.Close()
Transport
httptest
CORS
Cookie 安全属性
文件上传安全
pprof
短链接唯一性
访问计数并发安全
```

如果这些都能结合项目讲清楚，`net/http` 相关面试基本就稳了。
