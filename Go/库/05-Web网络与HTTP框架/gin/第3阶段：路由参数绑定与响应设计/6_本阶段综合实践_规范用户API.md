# 6. 综合实践：规范用户 API

本节目标：把本阶段学到的 RESTful 路由、参数绑定、字段校验、统一响应、错误码、分页筛选和状态码选择整合到用户 API 中。

这不是新知识点，而是一次收口练习。你要把前面每一节拆开的能力组合起来，形成一个更像真实项目的用户模块。

---

## 一、最终接口清单

本节最终实现：

```text
GET    /api/v1/users
POST   /api/v1/users
GET    /api/v1/users/:id
PUT    /api/v1/users/:id
PATCH  /api/v1/users/:id/status
DELETE /api/v1/users/:id
```

其中：

```text
GET /users
支持分页、状态筛选和关键词搜索。

POST /users
创建用户，校验 username、email、age、status。

GET /users/:id
查询用户详情，id 必须是数字。

PUT /users/:id
整体更新用户。

PATCH /users/:id/status
只修改用户状态。

DELETE /users/:id
删除用户。
```

---

## 二、建议文件结构

如果你还在单文件阶段，可以先写在 `main.go` 中。

但建议至少拆出：

```text
internal/response/response.go
internal/response/code.go
internal/response/validation.go
```

本阶段重点是接口规范，不强制完成完整分层。完整分层会在第5阶段展开。

---

## 三、用户结构体

```go
type User struct {
    ID        uint      `json:"id"`
    Username  string    `json:"username"`
    Email     string    `json:"email"`
    Age       int       `json:"age"`
    Status    string    `json:"status"`
    CreatedAt time.Time `json:"created_at"`
}
```

初始化数据：

```go
var users = map[uint]User{
    1: {ID: 1, Username: "alice", Email: "alice@example.com", Age: 20, Status: "active", CreatedAt: time.Now()},
    2: {ID: 2, Username: "bob", Email: "bob@example.com", Age: 22, Status: "disabled", CreatedAt: time.Now()},
}

var nextID uint = 3
```

需要导入：

```go
import "time"
```

---

## 四、请求结构体

创建用户：

```go
type CreateUserRequest struct {
    Username string `json:"username" binding:"required,min=2,max=32"`
    Email    string `json:"email" binding:"required,email"`
    Age      int    `json:"age" binding:"gte=1,lte=120"`
    Status   string `json:"status" binding:"omitempty,oneof=active disabled"`
}
```

更新用户：

```go
type UpdateUserRequest struct {
    Username string `json:"username" binding:"required,min=2,max=32"`
    Email    string `json:"email" binding:"required,email"`
    Age      int    `json:"age" binding:"gte=1,lte=120"`
    Status   string `json:"status" binding:"required,oneof=active disabled"`
}
```

修改状态：

```go
type UpdateUserStatusRequest struct {
    Status string `json:"status" binding:"required,oneof=active disabled"`
}
```

列表查询：

```go
type ListUsersRequest struct {
    Page     int    `form:"page" binding:"omitempty,gte=1"`
    PageSize int    `form:"page_size" binding:"omitempty,gte=1,lte=100"`
    Status   string `form:"status" binding:"omitempty,oneof=active disabled"`
    Keyword  string `form:"keyword" binding:"omitempty,max=50"`
}
```

---

## 五、公共 ID 解析函数

多个接口都要解析路径 ID，可以抽函数：

```go
func parseID(c *gin.Context) (uint, bool) {
    id, err := strconv.ParseUint(c.Param("id"), 10, 64)
    if err != nil {
        response.Fail(c, http.StatusBadRequest, response.CodeInvalidRequest, "invalid user id")
        return 0, false
    }
    return uint(id), true
}
```

使用：

```go
id, ok := parseID(c)
if !ok {
    return
}
```

这里返回 bool，是为了让 handler 可以在解析失败后直接结束。

---

## 六、创建用户

```go
func createUser(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.FailWithData(c, http.StatusBadRequest, response.CodeInvalidRequest, "invalid request", response.FormatValidationError(err))
        return
    }

    for _, user := range users {
        if user.Email == req.Email {
            response.Fail(c, http.StatusConflict, response.CodeConflict, "email already exists")
            return
        }
    }

    status := req.Status
    if status == "" {
        status = "active"
    }

    user := User{
        ID:        nextID,
        Username:  req.Username,
        Email:     req.Email,
        Age:       req.Age,
        Status:    status,
        CreatedAt: time.Now(),
    }

    users[user.ID] = user
    nextID++

    response.Created(c, user)
}
```

这里包含了本阶段多个重点：

```text
ShouldBindJSON
字段校验
邮箱冲突返回 409
默认状态
创建成功返回 201
统一响应
```

---

## 七、查询详情

```go
func getUser(c *gin.Context) {
    id, ok := parseID(c)
    if !ok {
        return
    }

    user, exists := users[id]
    if !exists {
        response.Fail(c, http.StatusNotFound, response.CodeNotFound, "user not found")
        return
    }

    response.OK(c, user)
}
```

测试：

```http
GET /api/v1/users/abc
```

应该返回 400。

```http
GET /api/v1/users/999
```

应该返回 404。

---

## 八、列表查询

```go
func listUsers(c *gin.Context) {
    var req ListUsersRequest
    if err := c.ShouldBindQuery(&req); err != nil {
        response.FailWithData(c, http.StatusBadRequest, response.CodeInvalidRequest, "invalid query", response.FormatValidationError(err))
        return
    }

    if req.Page == 0 {
        req.Page = 1
    }
    if req.PageSize == 0 {
        req.PageSize = 10
    }

    filtered := filterUsers(req)
    paged := paginateUsers(filtered, req.Page, req.PageSize)

    response.OK(c, response.PageResult{
        Items:    paged,
        Total:    int64(len(filtered)),
        Page:     req.Page,
        PageSize: req.PageSize,
    })
}
```

筛选函数和分页函数可以使用上一节实现。

---

## 九、更新用户

```go
func updateUser(c *gin.Context) {
    id, ok := parseID(c)
    if !ok {
        return
    }

    old, exists := users[id]
    if !exists {
        response.Fail(c, http.StatusNotFound, response.CodeNotFound, "user not found")
        return
    }

    var req UpdateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.FailWithData(c, http.StatusBadRequest, response.CodeInvalidRequest, "invalid request", response.FormatValidationError(err))
        return
    }

    for _, user := range users {
        if user.Email == req.Email && user.ID != id {
            response.Fail(c, http.StatusConflict, response.CodeConflict, "email already exists")
            return
        }
    }

    updated := User{
        ID:        id,
        Username:  req.Username,
        Email:     req.Email,
        Age:       req.Age,
        Status:    req.Status,
        CreatedAt: old.CreatedAt,
    }

    users[id] = updated
    response.OK(c, updated)
}
```

注意邮箱冲突要排除当前用户自己。

---

## 十、修改状态

```go
func updateUserStatus(c *gin.Context) {
    id, ok := parseID(c)
    if !ok {
        return
    }

    user, exists := users[id]
    if !exists {
        response.Fail(c, http.StatusNotFound, response.CodeNotFound, "user not found")
        return
    }

    var req UpdateUserStatusRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.FailWithData(c, http.StatusBadRequest, response.CodeInvalidRequest, "invalid request", response.FormatValidationError(err))
        return
    }

    user.Status = req.Status
    users[id] = user

    response.OK(c, user)
}
```

这里使用 `PATCH`，因为只更新一个字段。

---

## 十一、删除用户

```go
func deleteUser(c *gin.Context) {
    id, ok := parseID(c)
    if !ok {
        return
    }

    if _, exists := users[id]; !exists {
        response.Fail(c, http.StatusNotFound, response.CodeNotFound, "user not found")
        return
    }

    delete(users, id)
    response.OK(c, gin.H{"deleted": true})
}
```

学习阶段用内存删除。数据库阶段会学习软删除。

---

## 十二、路由注册

```go
func main() {
    r := gin.Default()
    r.HandleMethodNotAllowed = true

    api := r.Group("/api/v1")
    {
        api.GET("/users", listUsers)
        api.POST("/users", createUser)
        api.GET("/users/:id", getUser)
        api.PUT("/users/:id", updateUser)
        api.PATCH("/users/:id/status", updateUserStatus)
        api.DELETE("/users/:id", deleteUser)
    }

    r.NoRoute(func(c *gin.Context) {
        response.Fail(c, http.StatusNotFound, response.CodeNotFound, "route not found")
    })

    r.NoMethod(func(c *gin.Context) {
        response.Fail(c, http.StatusMethodNotAllowed, response.CodeInvalidRequest, "method not allowed")
    })

    r.Run(":8080")
}
```

---

## 十三、验收清单

完成后逐项检查：

- `GET /api/v1/users` 返回分页结构。
- `GET /api/v1/users?page_size=1000` 返回 400。
- `POST /api/v1/users` 正常创建返回 201。
- 创建用户时邮箱格式错误返回 400。
- 创建用户时邮箱重复返回 409。
- `GET /api/v1/users/abc` 返回 400。
- `GET /api/v1/users/999` 返回 404。
- `PATCH /api/v1/users/1/status` 可以修改状态。
- 非法状态返回 400。
- 不存在路由返回统一 404。
- 方法不允许返回统一 405。

---

## 十四、复盘问题

完成后请回答：

- 哪些错误属于参数错误？
- 哪些错误属于业务冲突？
- 哪些错误属于资源不存在？
- 哪些代码重复了，可以抽函数？
- 当前用户模块如果接数据库，需要改哪一层？

如果你能回答这些问题，就说明本阶段不是只把代码跑通了，而是真的理解了接口规范。

