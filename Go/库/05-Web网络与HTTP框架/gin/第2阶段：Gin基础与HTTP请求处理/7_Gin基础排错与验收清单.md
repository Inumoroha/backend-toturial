# 7. Gin 基础排错与验收清单

本节目标：把第2阶段 Gin 基础中最容易出错的点整理成清单，帮助你确认自己不是“看懂了”，而是真的能写、能测、能排错。

第2阶段的重点不是复杂项目结构，而是建立 Gin 的基本手感：

- 怎么启动 Gin 服务。
- `gin.Default()` 和 `gin.New()` 有什么区别。
- 路由如何注册。
- API 版本分组如何组织。
- 路径参数、查询参数、Header、JSON body 分别怎么取。
- 如何完成一个内存版 CRUD。

---

## 一、Gin 安装检查

安装：

```bash
go get github.com/gin-gonic/gin
```

检查：

```bash
go list -m github.com/gin-gonic/gin
go mod tidy
```

最小服务：

```go
package main

import "github.com/gin-gonic/gin"

func main() {
    r := gin.Default()

    r.GET("/ping", func(c *gin.Context) {
        c.JSON(200, gin.H{"message": "pong"})
    })

    r.Run(":8080")
}
```

验收：

- [ ] `go get` 成功。
- [ ] `go.mod` 中能看到 Gin。
- [ ] `/ping` 返回 JSON。

---

## 二、`gin.Default()` 与 `gin.New()` 检查

`gin.Default()` 自带：

```text
Logger
Recovery
```

`gin.New()` 是空引擎，需要自己注册中间件。

学习阶段可以用：

```go
r := gin.Default()
```

后续工程化阶段更常用：

```go
r := gin.New()
r.Use(gin.Logger())
r.Use(gin.Recovery())
```

自查问题：

- 你是否知道 panic 为什么没有打崩服务。
- 你是否知道请求日志从哪里来。
- 你是否能用 `gin.New()` 手动注册 Logger 和 Recovery。

---

## 三、路由注册排错

常见错误：

```text
404 page not found
```

排查：

- 请求方法是否正确。
- 路径是否正确。
- 是否漏了 `/api/v1` 前缀。
- 服务是否重启。
- 端口是否一致。

示例：

```go
api := r.Group("/api/v1")
api.GET("/users", listUsers)
```

正确请求：

```http
GET http://localhost:8080/api/v1/users
```

错误请求：

```http
GET http://localhost:8080/users
```

---

## 四、参数读取检查

路径参数：

```go
id := c.Param("id")
```

查询参数：

```go
page := c.DefaultQuery("page", "1")
```

Header：

```go
token := c.GetHeader("Authorization")
```

JSON body：

```go
var req CreateUserRequest
if err := c.ShouldBindJSON(&req); err != nil {
    c.JSON(400, gin.H{"message": "invalid request"})
    return
}
```

自查问题：

- ID 是否从路径参数取。
- 分页是否从查询参数取。
- token 是否从 Header 取。
- 创建用户是否从 JSON body 取。
- JSON 请求是否带 `Content-Type: application/json`。

---

## 五、JSON 绑定排错

如果 `ShouldBindJSON` 报错，优先检查：

- JSON 是否合法。
- 字段名是否和 `json` tag 一致。
- 请求头是否是 `application/json`。
- 数字字段是否传了字符串。
- 请求体是否为空。

示例结构体：

```go
type CreateUserRequest struct {
    Username string `json:"username"`
    Email    string `json:"email"`
    Age      int    `json:"age"`
}
```

正确请求：

```http
POST http://localhost:8080/api/v1/users
Content-Type: application/json

{
  "username": "alice",
  "email": "alice@example.com",
  "age": 20
}
```

错误请求：

```json
{
  "username": "alice",
  "age": "twenty"
}
```

---

## 六、内存 CRUD 检查

第2阶段内存版用户 CRUD 至少应该支持：

```text
GET    /api/v1/users
POST   /api/v1/users
GET    /api/v1/users/:id
PUT    /api/v1/users/:id
DELETE /api/v1/users/:id
```

内存存储示例：

```go
var (
    users  = map[uint]User{}
    nextID uint = 1
)
```

创建时：

```go
user.ID = nextID
nextID++
users[user.ID] = user
```

注意：内存数据重启后会丢失。这不是 bug，而是第2阶段为了聚焦 HTTP 和 Gin 基础。

---

## 七、接口测试顺序

建议按顺序测试：

```http
### list empty users
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
```

期望：

- 创建前列表为空。
- 创建后能查到用户。
- 更新后字段改变。
- 删除后再次查询返回 404。

---

## 八、常见问题

### 1. 每次重启用户都没了

正常。第2阶段用的是内存 map。第6阶段会接 GORM 和数据库。

### 2. 删除用户后列表还有

检查是否真的执行了：

```go
delete(users, id)
```

也检查 ID 类型转换是否正确。

### 3. 路径参数 ID 转换失败

路径参数是字符串，需要转换：

```go
id64, err := strconv.ParseUint(c.Param("id"), 10, 64)
```

转换失败应返回 `400`。

---

## 九、最终验收清单

完成第2阶段后，请确认：

- [ ] 能启动 Gin 服务。
- [ ] 能解释 `gin.Default()` 和 `gin.New()` 的区别。
- [ ] 能注册 `/api/v1` 路由组。
- [ ] 能读取路径参数。
- [ ] 能读取查询参数。
- [ ] 能读取 Header。
- [ ] 能绑定 JSON 请求体。
- [ ] 能完成内存版用户 CRUD。
- [ ] 能用请求工具验证成功和失败场景。
- [ ] 知道内存数据重启丢失是正常现象。

这些全部完成后，再进入第3阶段做参数校验、统一响应和更规范的 RESTful API。
