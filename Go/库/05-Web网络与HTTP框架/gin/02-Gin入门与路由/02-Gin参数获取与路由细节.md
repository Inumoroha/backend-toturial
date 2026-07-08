# 02-02 Gin 参数获取与路由细节

本节目标：把 Gin 中最常见的参数来源讲清楚。一个接口的问题，很多时候不是业务错了，而是参数根本没有按你以为的方式进入后端。

## 一、参数来自哪里

Gin 中常见参数来源：

- 路径参数：`/users/:id`
- 查询参数：`/users?page=1`
- JSON 请求体：`POST`、`PUT` 常用
- 表单参数：传统表单或文件上传常用
- Header：认证 token 常用

不同来源使用不同方法读取，不要混用。

## 二、路径参数

路由：

```go
r.GET("/users/:id", func(c *gin.Context) {
    id := c.Param("id")
    c.JSON(200, gin.H{"id": id})
})
```

请求：

```text
GET /users/123
```

返回：

```json
{
  "id": "123"
}
```

注意：`c.Param` 返回字符串。如果你需要数字，要手动转换：

```go
id, err := strconv.ParseUint(c.Param("id"), 10, 64)
```

## 三、查询参数

请求：

```text
GET /users?page=1&page_size=10&status=active
```

读取：

```go
page := c.DefaultQuery("page", "1")
status := c.Query("status")
```

如果参数很多，推荐绑定到结构体：

```go
type ListUsersRequest struct {
    Page     int    `form:"page"`
    PageSize int    `form:"page_size"`
    Status   string `form:"status"`
}

var req ListUsersRequest
if err := c.ShouldBindQuery(&req); err != nil {
    c.JSON(400, gin.H{"message": "invalid query"})
    return
}
```

## 四、JSON 请求体

请求：

```json
{
  "username": "alice",
  "email": "alice@example.com"
}
```

结构体：

```go
type CreateUserRequest struct {
    Username string `json:"username"`
    Email    string `json:"email"`
}
```

绑定：

```go
var req CreateUserRequest
if err := c.ShouldBindJSON(&req); err != nil {
    c.JSON(400, gin.H{"message": "invalid json"})
    return
}
```

常见错误：

- 请求头没有 `Content-Type: application/json`。
- JSON 少了逗号。
- 字段名和 `json` tag 不一致。
- 数字字段传了字符串。

## 五、Header 参数

JWT 常放在 Header：

```text
Authorization: Bearer token
```

读取：

```go
auth := c.GetHeader("Authorization")
```

不要把 token 放在普通查询参数里。查询参数容易进入浏览器历史、日志和监控系统。

## 六、路由组设计

推荐结构：

```go
r := gin.Default()

api := r.Group("/api/v1")
{
    api.POST("/auth/register", authHandler.Register)
    api.POST("/auth/login", authHandler.Login)

    private := api.Group("")
    private.Use(AuthMiddleware())
    {
        private.GET("/profile", userHandler.Profile)
        private.GET("/tasks", taskHandler.List)
    }
}
```

这样可以清楚地区分公开接口和登录后接口。

## 七、404 和 405

如果路径不存在，通常返回 404。

如果路径存在但方法不对，通常返回 405。Gin 可以设置：

```go
r.HandleMethodNotAllowed = true
```

并自定义：

```go
r.NoRoute(func(c *gin.Context) {
    c.JSON(404, gin.H{"message": "route not found"})
})

r.NoMethod(func(c *gin.Context) {
    c.JSON(405, gin.H{"message": "method not allowed"})
})
```

## 八、本节练习

实现一个搜索接口：

```text
GET /api/v1/users/search?keyword=alice&page=1&page_size=10
```

要求：

- 使用结构体绑定查询参数。
- `page` 默认值为 1。
- `page_size` 默认值为 10。
- `page_size` 最大不能超过 100。
- 返回统一 JSON。

## 九、验收清单

你应该能够回答：

- 路径参数和查询参数分别适合什么场景？
- 为什么查询参数绑定使用 `form` tag？
- `c.Query` 和 `c.DefaultQuery` 有什么区别？
- token 为什么常放在 Authorization header？
- Gin 中如何自定义 404 和 405？

