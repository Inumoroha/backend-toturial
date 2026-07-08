# 2. HTTP 基础概念

本节目标：理解 HTTP 请求和响应的基本结构。学习 `net/http` 时，你写的 `*http.Request` 对应 HTTP 请求，`http.ResponseWriter` 对应 HTTP 响应。

---

## 一、HTTP 是什么

HTTP 是客户端和服务端之间传输数据的协议。

你可以先把它理解成一种约定：

```text
客户端按照某种格式发请求。
服务端按照某种格式回响应。
双方都遵守这个格式，就能通信。
```

浏览器访问网页、前端调用后端 API、后端调用第三方服务，大多数都离不开 HTTP。

---

## 二、一次 HTTP 请求长什么样

一个请求通常包含：

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
Authorization: Bearer abc123

{"title":"learn net/http"}
```

拆开看：

- `POST` 是请求方法。
- `/todos` 是路径。
- `source=web` 是 Query 参数。
- `Host`、`Content-Type`、`Authorization` 是 Header。
- `{"title":"learn net/http"}` 是 Body。

在 Go 里，它们大致对应：

```go
r.Method
r.URL.Path
r.URL.Query()
r.Header
r.Body
```

---

## 三、一次 HTTP 响应长什么样

响应通常包含：

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

{"id":1,"title":"learn net/http"}
```

拆开看：

- `201 Created` 表示创建成功。
- `Content-Type: application/json` 表示响应体是 JSON。
- Body 是真正返回给客户端的数据。

在 Go 里，你主要通过 `http.ResponseWriter` 写响应：

```go
w.Header().Set("Content-Type", "application/json")
w.WriteHeader(http.StatusCreated)
w.Write([]byte(`{"id":1}`))
```

---

## 四、Method：请求想做什么

常见 HTTP Method：

- `GET`：获取资源。
- `POST`：创建资源或提交动作。
- `PUT`：整体替换资源。
- `PATCH`：局部更新资源。
- `DELETE`：删除资源。

例子：

```text
GET    /todos       获取 Todo 列表
POST   /todos       创建 Todo
GET    /todos/1     获取 ID 为 1 的 Todo
PATCH  /todos/1     修改 ID 为 1 的 Todo
DELETE /todos/1     删除 ID 为 1 的 Todo
```

Method 不是装饰，它表达的是接口语义。后端工程师写 API 时，要尽量让方法和动作匹配。

---

## 五、Path 与 Query 的区别

Path 表示资源位置：

```text
/users/10
/todos/3
/articles/go-net-http
```

Query 表示过滤、搜索、分页、排序等附加条件：

```text
/todos?done=false
/articles?page=2&page_size=20
/search?q=golang
```

经验规则：

- 资源身份放 Path。
- 筛选条件放 Query。
- 复杂创建和更新数据放 Body。

---

## 六、Header 放什么

Header 是请求或响应的元信息。

常见请求 Header：

- `Content-Type`：请求体是什么格式。
- `Authorization`：认证信息。
- `User-Agent`：客户端信息。
- `Cookie`：浏览器携带的 Cookie。

常见响应 Header：

- `Content-Type`：响应体是什么格式。
- `Set-Cookie`：服务端设置 Cookie。
- `Location`：重定向地址。
- `Cache-Control`：缓存策略。

Header 适合放“描述这次 HTTP 消息的信息”，不适合放大块业务数据。

---

## 七、Body 放什么

Body 是请求或响应的主体数据。

常见格式：

- JSON：API 最常见。
- Form：传统表单。
- Multipart：文件上传。
- HTML：网页响应。
- Plain Text：纯文本。

JSON API 常见请求：

```json
{
  "title": "learn net/http",
  "done": false
}
```

对应 Header 通常要设置：

```text
Content-Type: application/json
```

如果客户端没有正确设置 `Content-Type`，服务端有时仍能读取 Body，但语义不清楚，排查问题也更难。

---

## 八、状态码

状态码告诉客户端请求结果。

常见状态码：

- `200 OK`：请求成功。
- `201 Created`：资源创建成功。
- `204 No Content`：成功但没有响应体。
- `301 Moved Permanently`：永久重定向。
- `302 Found`：临时重定向。
- `400 Bad Request`：请求格式或参数错误。
- `401 Unauthorized`：没有认证。
- `403 Forbidden`：认证了但没有权限。
- `404 Not Found`：资源不存在。
- `405 Method Not Allowed`：方法不允许。
- `409 Conflict`：资源冲突。
- `500 Internal Server Error`：服务端内部错误。

一个常见错误是：业务失败了，但响应状态码仍然是 200。

例如：

```json
{"error":"todo not found"}
```

如果它的 HTTP 状态码是 200，客户端就很难统一判断成功失败。正确做法通常是返回：

```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{"error":"todo not found"}
```

---

## 九、HTTP 是无状态的

HTTP 本身是无状态的。也就是说，每一次请求在协议层面都是独立的。

服务端不会天然记得：

- 上一次是谁访问了。
- 这个用户是否登录过。
- 之前请求做了什么。

真实系统通过这些方式补充状态：

- Cookie。
- Session。
- Token。
- 数据库。
- 缓存。

理解无状态很重要，因为它解释了为什么很多接口都需要携带认证信息。

---

## 十、本节检查点

请确认你能解释：

- Method、Path、Query、Header、Body 分别是什么。
- `GET /users/1` 和 `GET /users?id=1` 的表达差异。
- 为什么创建资源通常返回 `201`。
- 为什么参数错误应该返回 `400`。
- 为什么未登录和无权限分别是 `401` 与 `403`。

如果这些概念清楚了，学习 `net/http` 的 API 会自然很多。

