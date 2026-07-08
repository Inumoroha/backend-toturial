# 6. 内存版用户 CRUD 完整实践

本节目标：把前面学到的路由、路径参数、查询参数和 JSON 请求体组合起来，完成一个内存版用户 CRUD。

这一节暂时不接数据库。重点是熟练 Gin 的 HTTP 处理流程。

---

## 一、接口设计

本节实现：

```text
GET    /api/v1/users
GET    /api/v1/users/:id
POST   /api/v1/users
PUT    /api/v1/users/:id
DELETE /api/v1/users/:id
```

数据暂时保存在：

```go
map[uint]User
```

服务重启后数据会丢失，这是正常的。后面数据库阶段会解决。

---

## 二、完整代码

`main.go`：

```go
package main

import (
    "net/http"
    "strconv"

    "github.com/gin-gonic/gin"
)

type User struct {
    ID       uint   `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email"`
    Age      int    `json:"age"`
}

type CreateUserRequest struct {
    Username string `json:"username"`
    Email    string `json:"email"`
    Age      int    `json:"age"`
}

var users = map[uint]User{
    1: {ID: 1, Username: "alice", Email: "alice@example.com", Age: 20},
    2: {ID: 2, Username: "bob", Email: "bob@example.com", Age: 22},
}

var nextID uint = 3

func main() {
    r := gin.Default()

    api := r.Group("/api/v1")
    {
        api.GET("/users", listUsers)
        api.GET("/users/:id", getUser)
        api.POST("/users", createUser)
        api.PUT("/users/:id", updateUser)
        api.DELETE("/users/:id", deleteUser)
    }

    r.Run(":8080")
}

func listUsers(c *gin.Context) {
    result := make([]User, 0, len(users))
    for _, user := range users {
        result = append(result, user)
    }

    c.JSON(http.StatusOK, gin.H{"data": result})
}

func getUser(c *gin.Context) {
    id, err := parseID(c.Param("id"))
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"message": "invalid user id"})
        return
    }

    user, ok := users[id]
    if !ok {
        c.JSON(http.StatusNotFound, gin.H{"message": "user not found"})
        return
    }

    c.JSON(http.StatusOK, gin.H{"data": user})
}

func createUser(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"message": "invalid request"})
        return
    }

    user := User{
        ID:       nextID,
        Username: req.Username,
        Email:    req.Email,
        Age:      req.Age,
    }
    users[user.ID] = user
    nextID++

    c.JSON(http.StatusCreated, gin.H{"data": user})
}

func updateUser(c *gin.Context) {
    id, err := parseID(c.Param("id"))
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"message": "invalid user id"})
        return
    }

    if _, ok := users[id]; !ok {
        c.JSON(http.StatusNotFound, gin.H{"message": "user not found"})
        return
    }

    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"message": "invalid request"})
        return
    }

    user := User{
        ID:       id,
        Username: req.Username,
        Email:    req.Email,
        Age:      req.Age,
    }
    users[id] = user

    c.JSON(http.StatusOK, gin.H{"data": user})
}

func deleteUser(c *gin.Context) {
    id, err := parseID(c.Param("id"))
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"message": "invalid user id"})
        return
    }

    if _, ok := users[id]; !ok {
        c.JSON(http.StatusNotFound, gin.H{"message": "user not found"})
        return
    }

    delete(users, id)
    c.JSON(http.StatusOK, gin.H{"message": "deleted"})
}

func parseID(raw string) (uint, error) {
    id, err := strconv.ParseUint(raw, 10, 64)
    if err != nil {
        return 0, err
    }
    return uint(id), nil
}
```

---

## 三、代码重点解释

### 1. 为什么用 map

```go
map[uint]User
```

可以通过 ID 快速找到用户。

例如：

```go
user, ok := users[id]
```

如果 `ok` 是 false，说明用户不存在。

### 2. 为什么有 nextID

创建新用户时需要生成新 ID：

```go
ID: nextID
```

创建后递增：

```go
nextID++
```

这只是学习用。真实项目中 ID 通常由数据库生成。

### 3. 为什么抽 parseID

多个接口都要把路径参数转数字，所以抽成函数：

```go
func parseID(raw string) (uint, error)
```

这能减少重复代码。

---

## 四、requests.http

```http
### list users
GET http://localhost:8080/api/v1/users

### get user
GET http://localhost:8080/api/v1/users/1

### create user
POST http://localhost:8080/api/v1/users
Content-Type: application/json

{
  "username": "cindy",
  "email": "cindy@example.com",
  "age": 21
}

### update user
PUT http://localhost:8080/api/v1/users/1
Content-Type: application/json

{
  "username": "alice-new",
  "email": "alice_new@example.com",
  "age": 23
}

### delete user
DELETE http://localhost:8080/api/v1/users/1
```

---

## 五、常见问题

### 1. 删除后数据又回来了

如果你重启服务，内存 map 会恢复成代码里的初始值。这是正常的。

数据库阶段会解决数据持久化。

### 2. 用户列表顺序不固定

Go map 遍历顺序不保证稳定。

学习阶段可以接受。后面数据库查询时使用 `ORDER BY`。

### 3. 并发读写 map 有风险

当前只是本地学习。真实服务中并发读写 map 可能 panic。

后面会用数据库替代。

---

## 六、本节练习

在用户中增加字段：

```go
Status string `json:"status"`
```

要求：

- 创建用户时默认 `active`。
- 列表接口支持 `?status=active` 筛选。
- 增加接口 `PATCH /api/v1/users/:id/status`。

---

## 七、本节验收

你应该能够回答：

- 为什么本阶段先用内存 map？
- `nextID` 起什么作用？
- 用户不存在为什么返回 404？
- 创建成功为什么返回 201？
- 为什么 map 列表顺序不固定？

