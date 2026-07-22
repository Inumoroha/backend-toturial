# 3. 路由注册与 API 版本分组

本节目标：掌握 Gin 中 GET、POST、PUT、PATCH、DELETE 路由注册方式，并学会使用 `/api/v1` 组织接口。

---

## 一、什么是路由

路由就是：

```text
HTTP 方法 + 请求路径 -> 处理函数
```

例如：

```text
GET /ping -> pingHandler
POST /users -> createUserHandler
```

在 Gin 中写成：

```go
r.GET("/ping", pingHandler)
r.POST("/users", createUserHandler)
```

---

## 二、注册基本路由

示例：

```go
func main() {
    r := gin.Default()

    r.GET("/ping", ping)
    r.GET("/users", listUsers)
    r.POST("/users", createUser)
    r.PUT("/users/:id", updateUser)
    r.PATCH("/users/:id/status", updateUserStatus)
    r.DELETE("/users/:id", deleteUser)

    r.Run(":8080")
}
```

handler 可以先写成：

```go
func ping(c *gin.Context) {
    c.JSON(200, gin.H{"message": "pong"})
}
```

---

## 三、为什么要使用 `/api/v1`

真实项目中，不建议所有接口都挂在根路径：

```text
/users
/tasks
/orders
```

更推荐：

```text
/api/v1/users
/api/v1/tasks
/api/v1/orders
```

好处：

- 明确这是 API，不是页面路由。
- 方便未来升级版本。
- 方便网关或 Nginx 做统一转发。

例如将来有不兼容升级，可以增加：

```text
/api/v2/users
```

---

## 四、使用路由组

Gin 路由组：

```go
api := r.Group("/api/v1")
{
    api.GET("/users", listUsers)
    api.POST("/users", createUser)
    api.GET("/users/:id", getUser)
}
```

实际路径是：

```text
GET /api/v1/users
POST /api/v1/users
GET /api/v1/users/:id
```

路由组还能统一挂载中间件，后面认证阶段会用到。

---

## 五、公开路由和私有路由

后续项目常见结构：

```go
api := r.Group("/api/v1")

api.POST("/auth/register", register)
api.POST("/auth/login", login)

private := api.Group("")
private.Use(AuthMiddleware())
{
    private.GET("/profile", profile)
    private.GET("/tasks", listTasks)
}
```

含义：

- 注册、登录不需要 token。
- 个人信息、任务列表需要 token。

先记住这个结构，后面做 JWT 时会反复使用。

---

## 六、RESTful 路由命名

推荐：

```text
GET    /api/v1/users
POST   /api/v1/users
GET    /api/v1/users/:id
PUT    /api/v1/users/:id
DELETE /api/v1/users/:id
```

不推荐：

```text
GET  /api/v1/getUsers
POST /api/v1/createUser
POST /api/v1/deleteUser
```

原因是动作已经由 HTTP 方法表达，路径尽量表示资源。

---

## 七、本节练习

创建这些路由，先返回固定 JSON：

```text
GET    /api/v1/users
POST   /api/v1/users
GET    /api/v1/users/:id
PUT    /api/v1/users/:id
DELETE /api/v1/users/:id
```

每个接口返回不同 message，例如：

```json
{
  "message": "list users"
}
```

然后写入 `requests.http` 测试。

---

## 八、本节验收

你应该能够回答：

- 路由由哪两部分组成？
- 为什么推荐 `/api/v1`？
- `r.Group("/api/v1")` 有什么作用？
- 公开路由和私有路由有什么区别？
- RESTful 路由为什么路径中尽量不用动词？


