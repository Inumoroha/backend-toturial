# 8. 本阶段综合实践：Gin 内存用户 API 验收

本节目标：用 Gin 完成一个内存版用户 API，并用请求集合验证完整 CRUD 行为。

第2阶段的最终成果不是只会写 `/ping`，而是能用 Gin 独立写出一个小型资源模块。这个模块暂时不接数据库、不做复杂校验，重点是 HTTP、路由、参数和 JSON。

---

## 一、最终接口

```text
GET    /api/v1/users
POST   /api/v1/users
GET    /api/v1/users/:id
PUT    /api/v1/users/:id
DELETE /api/v1/users/:id
```

用户字段：

```text
id
username
email
age
```

内存数据重启后丢失，这是本阶段的正常设计。

---

## 二、项目结构

```text
gin-memory-users/
├── go.mod
├── main.go
├── requests.http
└── README.md
```

初始化：

```bash
mkdir gin-memory-users
cd gin-memory-users
go mod init gin-memory-users
go get github.com/gin-gonic/gin
```

---

## 三、核心代码

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

var (
    users  = map[uint]User{}
    nextID uint = 1
)

func main() {
    r := gin.Default()

    api := r.Group("/api/v1")
    {
        api.GET("/users", listUsers)
        api.POST("/users", createUser)
        api.GET("/users/:id", getUser)
        api.PUT("/users/:id", updateUser)
        api.DELETE("/users/:id", deleteUser)
    }

    r.Run(":8080")
}
```

先不要急着分层。第2阶段重点是 Gin 基础。第5阶段会再拆 handler、service、repository。

---

## 四、实现列表和创建

```go
func listUsers(c *gin.Context) {
    result := make([]User, 0, len(users))
    for _, user := range users {
        result = append(result, user)
    }

    c.JSON(http.StatusOK, gin.H{
        "data": result,
    })
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
```

注意：这里还没有 binding 校验，所以空 username 也可能通过。第3阶段会专门处理参数校验。

---

## 五、实现详情、更新、删除

```go
func parseID(c *gin.Context) (uint, bool) {
    id64, err := strconv.ParseUint(c.Param("id"), 10, 64)
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"message": "invalid id"})
        return 0, false
    }
    return uint(id64), true
}

func getUser(c *gin.Context) {
    id, ok := parseID(c)
    if !ok {
        return
    }

    user, exists := users[id]
    if !exists {
        c.JSON(http.StatusNotFound, gin.H{"message": "user not found"})
        return
    }

    c.JSON(http.StatusOK, gin.H{"data": user})
}

func updateUser(c *gin.Context) {
    id, ok := parseID(c)
    if !ok {
        return
    }

    _, exists := users[id]
    if !exists {
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
    id, ok := parseID(c)
    if !ok {
        return
    }

    if _, exists := users[id]; !exists {
        c.JSON(http.StatusNotFound, gin.H{"message": "user not found"})
        return
    }

    delete(users, id)
    c.JSON(http.StatusOK, gin.H{"message": "deleted"})
}
```

---

## 六、请求集合

创建 `requests.http`：

```http
### list users
GET http://localhost:8080/api/v1/users

### create user
POST http://localhost:8080/api/v1/users
Content-Type: application/json

{
  "username": "alice",
  "email": "alice@example.com",
  "age": 20
}

### get user
GET http://localhost:8080/api/v1/users/1

### update user
PUT http://localhost:8080/api/v1/users/1
Content-Type: application/json

{
  "username": "alice-new",
  "email": "alice-new@example.com",
  "age": 21
}

### delete user
DELETE http://localhost:8080/api/v1/users/1

### get deleted user
GET http://localhost:8080/api/v1/users/1

### invalid id
GET http://localhost:8080/api/v1/users/abc
```

---

## 七、测试顺序

建议顺序：

```text
1. 查询列表，确认为空。
2. 创建用户，复制返回 id。
3. 查询详情。
4. 更新用户。
5. 再次查询详情。
6. 删除用户。
7. 查询已删除用户。
8. 请求非法 id。
```

期望：

- 创建返回 `201`。
- 查询存在用户返回 `200`。
- 查询不存在用户返回 `404`。
- 非法 ID 返回 `400`。

---

## 八、README

写一个最小 README：

```md
# Gin Memory Users

第2阶段综合实践：使用 Gin 实现内存版用户 CRUD。

## 启动

```bash
go run main.go
```

## 接口

- `GET /api/v1/users`
- `POST /api/v1/users`
- `GET /api/v1/users/:id`
- `PUT /api/v1/users/:id`
- `DELETE /api/v1/users/:id`

## 注意

数据保存在内存中，服务重启后会丢失。
```

---

## 九、常见问题

### 1. 创建用户返回 400

检查 JSON 是否合法，是否有 `Content-Type: application/json`。

### 2. 查询用户一直 404

检查是否先创建用户，服务是否重启过，ID 是否正确。

### 3. 更新后 ID 变了

更新时应该保留路径中的 ID，不要使用请求体里的 ID。

### 4. 删除后列表还有数据

检查是否调用 `delete(users, id)`，以及删除的 ID 是否正确。

---

## 十、最终验收

请确认：

- [ ] 能独立创建 Gin 项目。
- [ ] 能注册 API 路由组。
- [ ] 能实现列表、创建、详情、更新、删除。
- [ ] 能解析路径参数 ID。
- [ ] 能绑定 JSON 请求体。
- [ ] 能返回 JSON 响应。
- [ ] 能区分 `400`、`404`、`201`、`200`。
- [ ] 能用请求集合完成完整 CRUD 验收。

完成后再进入第3阶段，你会更容易理解为什么需要参数校验、统一响应和更规范的错误码。

