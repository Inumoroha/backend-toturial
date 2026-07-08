# 4. 路径参数、查询参数与 Header

本节目标：掌握 Gin 中三种常见请求数据来源：路径参数、查询参数和 Header。

---

## 一、三种参数分别是什么

路径参数：

```text
GET /api/v1/users/123
```

其中 `123` 是用户 ID。

查询参数：

```text
GET /api/v1/users?page=1&page_size=10&status=active
```

其中 `page`、`page_size`、`status` 是筛选条件。

Header：

```text
Authorization: Bearer token
```

常用于认证、追踪、内容协商。

---

## 二、读取路径参数

路由：

```go
api.GET("/users/:id", getUser)
```

handler：

```go
func getUser(c *gin.Context) {
    id := c.Param("id")
    c.JSON(200, gin.H{
        "id": id,
    })
}
```

请求：

```http
GET http://localhost:8080/api/v1/users/123
```

响应：

```json
{
  "id": "123"
}
```

注意：`c.Param("id")` 返回的是字符串。

---

## 三、路径参数转数字

通常 ID 需要转成数字：

```go
id, err := strconv.ParseUint(c.Param("id"), 10, 64)
if err != nil {
    c.JSON(400, gin.H{"message": "invalid user id"})
    return
}
```

参数解释：

- 第一个参数：要转换的字符串。
- `10`：十进制。
- `64`：转换成 uint64。

如果请求：

```text
/users/abc
```

就会返回 400。

---

## 四、读取查询参数

请求：

```http
GET http://localhost:8080/api/v1/users?page=1&page_size=10
```

handler：

```go
func listUsers(c *gin.Context) {
    page := c.DefaultQuery("page", "1")
    pageSize := c.DefaultQuery("page_size", "10")

    c.JSON(200, gin.H{
        "page": page,
        "page_size": pageSize,
    })
}
```

`c.Query("page")`：没有参数时返回空字符串。

`c.DefaultQuery("page", "1")`：没有参数时返回默认值。

---

## 五、使用结构体绑定查询参数

参数多时推荐：

```go
type ListUsersRequest struct {
    Page     int    `form:"page"`
    PageSize int    `form:"page_size"`
    Status   string `form:"status"`
}

func listUsers(c *gin.Context) {
    var req ListUsersRequest
    if err := c.ShouldBindQuery(&req); err != nil {
        c.JSON(400, gin.H{"message": "invalid query"})
        return
    }

    if req.Page == 0 {
        req.Page = 1
    }
    if req.PageSize == 0 {
        req.PageSize = 10
    }

    c.JSON(200, req)
}
```

注意查询参数使用 `form` tag，不是 `json` tag。

---

## 六、读取 Header

读取 Authorization：

```go
auth := c.GetHeader("Authorization")
```

示例：

```go
func profile(c *gin.Context) {
    auth := c.GetHeader("Authorization")
    if auth == "" {
        c.JSON(401, gin.H{"message": "missing authorization"})
        return
    }

    c.JSON(200, gin.H{"authorization": auth})
}
```

请求：

```http
GET http://localhost:8080/api/v1/profile
Authorization: Bearer demo-token
```

---

## 七、什么时候用哪种参数

建议：

```text
资源 ID：路径参数
筛选分页：查询参数
创建更新数据：JSON 请求体
认证信息：Header
```

例子：

```text
GET /api/v1/tasks/12
GET /api/v1/tasks?page=1&status=todo
POST /api/v1/tasks
Authorization: Bearer token
```

---

## 八、本节练习

实现：

```text
GET /api/v1/users/:id
GET /api/v1/users?page=1&page_size=10&status=active
GET /api/v1/profile
```

要求：

- `id` 必须转数字，失败返回 400。
- `page` 和 `page_size` 有默认值。
- `/profile` 读取 Authorization header。

---

## 九、本节验收

你应该能够回答：

- 路径参数适合什么场景？
- 查询参数适合什么场景？
- Header 适合什么场景？
- `c.Query` 和 `c.DefaultQuery` 有什么区别？
- 查询参数绑定为什么用 `form` tag？

