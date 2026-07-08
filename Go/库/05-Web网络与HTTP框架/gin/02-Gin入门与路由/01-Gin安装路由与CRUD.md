# 02-01 Gin 安装、路由与内存版 CRUD

本阶段目标：掌握 Gin 的基本开发方式，完成一个不依赖数据库的用户 CRUD API。

## 一、安装 Gin

在项目目录执行：

```bash
go get github.com/gin-gonic/gin
```

创建 `main.go`：

```go
package main

import "github.com/gin-gonic/gin"

func main() {
    r := gin.Default()

    r.GET("/ping", func(c *gin.Context) {
        c.JSON(200, gin.H{
            "message": "pong",
        })
    })

    r.Run(":8080")
}
```

运行：

```bash
go run main.go
```

访问：

```text
GET http://localhost:8080/ping
```

## 二、理解 `gin.Default()`

`gin.Default()` 会创建一个 Gin 引擎，并默认挂载两个中间件：

- Logger：打印请求日志。
- Recovery：从 panic 中恢复，避免服务直接崩溃。

如果你想完全自定义中间件，可以使用：

```go
r := gin.New()
```

初学阶段使用 `gin.Default()` 即可。

## 三、路由注册

Gin 使用方法名注册不同 HTTP 方法：

```go
r.GET("/users", listUsers)
r.POST("/users", createUser)
r.PUT("/users/:id", updateUser)
r.DELETE("/users/:id", deleteUser)
```

这里的 `:id` 是路径参数，例如：

```text
/users/123
```

可以通过：

```go
id := c.Param("id")
```

取到 `123`。

## 四、路由分组

真实项目通常会给 API 加版本号：

```go
api := r.Group("/api/v1")
{
    api.GET("/users", listUsers)
    api.POST("/users", createUser)
}
```

这样接口路径就是：

```text
GET /api/v1/users
POST /api/v1/users
```

以后升级到 v2 时，可以新建：

```go
v2 := r.Group("/api/v2")
```

## 五、实现内存版用户 CRUD

下面先使用内存 map 存储用户。这样可以专注学习 Gin，不被数据库干扰。

```go
package main

import (
    "net/http"
    "strconv"

    "github.com/gin-gonic/gin"
)

type User struct {
    ID       int    `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email"`
}

type CreateUserRequest struct {
    Username string `json:"username"`
    Email    string `json:"email"`
}

var (
    users = map[int]User{
        1: {ID: 1, Username: "alice", Email: "alice@example.com"},
        2: {ID: 2, Username: "bob", Email: "bob@example.com"},
    }
    nextID = 3
)

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

    c.JSON(http.StatusOK, gin.H{
        "data": result,
    })
}

func getUser(c *gin.Context) {
    id, err := strconv.Atoi(c.Param("id"))
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
    }
    users[user.ID] = user
    nextID++

    c.JSON(http.StatusCreated, gin.H{"data": user})
}

func updateUser(c *gin.Context) {
    id, err := strconv.Atoi(c.Param("id"))
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"message": "invalid user id"})
        return
    }

    _, ok := users[id]
    if !ok {
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
    }
    users[id] = user

    c.JSON(http.StatusOK, gin.H{"data": user})
}

func deleteUser(c *gin.Context) {
    id, err := strconv.Atoi(c.Param("id"))
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
```

## 六、测试接口

### 查询列表

```http
GET http://localhost:8080/api/v1/users
```

### 查询详情

```http
GET http://localhost:8080/api/v1/users/1
```

### 创建用户

```http
POST http://localhost:8080/api/v1/users
Content-Type: application/json

{
  "username": "cindy",
  "email": "cindy@example.com"
}
```

### 更新用户

```http
PUT http://localhost:8080/api/v1/users/1
Content-Type: application/json

{
  "username": "alice-new",
  "email": "alice_new@example.com"
}
```

### 删除用户

```http
DELETE http://localhost:8080/api/v1/users/1
```

## 七、常见问题

### 1. `ShouldBindJSON` 读不到数据

确认请求头带了：

```text
Content-Type: application/json
```

并确认请求体是合法 JSON。

### 2. 路由顺序

如果同时有：

```go
api.GET("/users/:id", getUser)
api.GET("/users/search", searchUsers)
```

要注意路由匹配冲突。一般更具体的路径要设计得清晰，例如：

```text
/users/search
/users/:id
```

并避免含义相近的路径混在一起。

### 3. 内存 map 不适合生产

当前 map 只是学习使用。真实服务重启后数据会丢失，并且并发读写 map 也有风险。后续会替换为数据库。

## 八、阶段练习

在现有代码基础上完成：

- 增加 `status` 字段，取值为 `active` 或 `disabled`。
- 查询列表时支持 `?status=active` 筛选。
- 创建用户时给默认状态 `active`。
- 增加接口 `PATCH /api/v1/users/:id/status` 修改用户状态。

## 九、验收清单

你应该能够回答：

- `gin.Default()` 做了什么？
- `c.Param`、`c.Query`、`c.ShouldBindJSON` 分别用于什么场景？
- 为什么推荐使用 `/api/v1`？
- `201 Created` 和 `200 OK` 的区别是什么？
- 为什么初学阶段可以先用内存 map？

