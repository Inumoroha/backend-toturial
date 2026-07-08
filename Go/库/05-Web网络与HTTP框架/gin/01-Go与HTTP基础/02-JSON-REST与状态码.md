# 01-02 JSON、REST 与 HTTP 状态码

本节目标：理解后端接口为什么通常返回 JSON，RESTful API 如何设计，以及常见 HTTP 状态码该怎么用。

## 一、为什么后端常用 JSON

JSON 的优点：

- 前端 JavaScript 原生支持。
- 结构清晰，适合表达对象和数组。
- 跨语言，Go、Java、Python、Node.js 都能处理。
- 可读性比二进制协议好，适合学习和调试。

Go 中结构体转 JSON：

```go
type User struct {
    ID       int    `json:"id"`
    Username string `json:"username"`
}
```

返回：

```json
{
  "id": 1,
  "username": "alice"
}
```

如果没有 `json` tag，Go 会使用结构体字段名，字段名通常是大写，不符合常见 API 风格。

## 二、请求体 JSON 怎么读

标准库写法：

```go
var req CreateUserRequest
if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
    http.Error(w, "invalid json", http.StatusBadRequest)
    return
}
```

Gin 写法：

```go
var req CreateUserRequest
if err := c.ShouldBindJSON(&req); err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"message": "invalid request"})
    return
}
```

Gin 并不是让 JSON 消失了，而是把重复代码封装好了。

## 三、RESTful API 基本规则

RESTful API 常用资源名表达业务对象：

```text
/users
/users/:id
/tasks
/tasks/:id
```

HTTP 方法表达动作：

```text
GET     查询
POST    创建
PUT     整体更新
PATCH   局部更新
DELETE  删除
```

推荐：

```text
GET /api/v1/users
POST /api/v1/users
GET /api/v1/users/1
PUT /api/v1/users/1
DELETE /api/v1/users/1
```

不推荐：

```text
GET /getUsers
POST /createUser
POST /deleteUser
```

原因是动词已经由 HTTP 方法表达了，路径里尽量使用名词。

## 四、常见状态码

### 200 OK

请求成功，常用于查询和更新成功。

### 201 Created

资源创建成功，常用于 `POST /users`。

### 204 No Content

请求成功但没有响应体，常用于删除成功。学习阶段也可以用 200 返回提示信息。

### 400 Bad Request

请求参数错误，例如 JSON 格式错误、字段校验失败。

### 401 Unauthorized

没有登录，或者 token 无效。

### 403 Forbidden

已经登录，但没有权限访问。

### 404 Not Found

资源不存在，例如用户 ID 不存在。

### 409 Conflict

资源冲突，例如邮箱已经被注册。

### 500 Internal Server Error

服务内部错误，例如数据库异常、程序 bug。

## 五、状态码和业务码的关系

建议同时使用 HTTP 状态码和业务错误码：

```json
{
  "code": 40001,
  "message": "invalid request"
}
```

HTTP 状态码给通用客户端和网关看，业务码给前端和业务逻辑看。

例如：

- HTTP 400 + 业务码 40001：参数错误。
- HTTP 401 + 业务码 40101：未登录。
- HTTP 409 + 业务码 40901：邮箱重复。

## 六、接口设计练习

请为“文章系统”设计接口：

- 创建文章。
- 查询文章列表。
- 查询文章详情。
- 更新文章。
- 删除文章。
- 发布文章。

参考答案：

```text
POST   /api/v1/articles
GET    /api/v1/articles
GET    /api/v1/articles/:id
PUT    /api/v1/articles/:id
DELETE /api/v1/articles/:id
PATCH  /api/v1/articles/:id/status
```

发布文章属于局部更新状态，所以可以使用 PATCH。

## 七、验收清单

你应该能够回答：

- 为什么 API 路径尽量使用名词？
- POST 和 PUT 有什么区别？
- 401 和 403 有什么区别？
- 400 和 500 应该如何区分？
- 为什么业务错误码不能完全替代 HTTP 状态码？

