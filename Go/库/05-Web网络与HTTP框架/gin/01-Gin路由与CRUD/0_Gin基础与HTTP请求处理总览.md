# 0. Gin 基础与 HTTP 请求处理总览

本阶段目标：从标准库 HTTP 服务过渡到 Gin，掌握 Gin 的启动方式、路由注册、请求参数读取、JSON 响应和内存版 CRUD。

第一阶段你已经知道标准库中一次请求大概这样处理：

```text
http.HandleFunc 注册路由
handler 读取 *http.Request
handler 通过 http.ResponseWriter 写响应
```

Gin 做的事情不是改变 HTTP 本质，而是让这些操作更简洁：

```text
router.GET 注册路由
*gin.Context 统一封装请求和响应
c.JSON 快速返回 JSON
c.Param / c.Query / c.ShouldBindJSON 读取参数
```

---

## 一、本阶段要解决的问题

本阶段解决这些问题：

- 如何安装 Gin。
- `gin.Default()` 和 `gin.New()` 有什么区别。
- 如何注册 GET、POST、PUT、DELETE 路由。
- 如何使用路由组 `/api/v1`。
- 如何读取路径参数、查询参数和 JSON 请求体。
- 如何返回 JSON 响应。
- 如何实现一个不依赖数据库的用户 CRUD。

---

## 二、本阶段统一练习接口

本阶段围绕用户资源练习：

```text
GET    /ping
GET    /api/v1/users
GET    /api/v1/users/:id
POST   /api/v1/users
PUT    /api/v1/users/:id
DELETE /api/v1/users/:id
```

暂时不接数据库，使用内存 map 保存数据。

这样做的原因是：先把 Gin 的 HTTP 处理学清楚，再去接数据库。不要一开始把路由、参数、GORM、认证混在一起。

---

## 三、本阶段推荐文件结构

本阶段先保持简单：

```text
gin-lab/
  go.mod
  go.sum
  main.go
  requests.http
```

等进入项目分层阶段，再拆成 `handler`、`service`、`repository`。

初学阶段过早拆目录，反而容易看不清 Gin 的核心流程。

---

## 四、本阶段学习顺序

建议按下面顺序：

```text
安装 Gin
→ 启动第一个 Gin 服务
→ 理解 gin.Default
→ 注册路由和路由组
→ 读取参数
→ 实现内存版 CRUD
→ 使用 requests.http 完整测试
```

---

## 五、本阶段完成标准

完成本阶段后，你应该能够：

- 独立创建 Gin 服务。
- 写出 `/ping` 接口。
- 使用 `/api/v1` 路由组。
- 读取 `:id` 路径参数。
- 读取 `?page=1` 查询参数。
- 使用 `ShouldBindJSON` 读取请求体。
- 实现用户 CRUD。
- 能根据 400、404、405 判断常见问题。

---

## 融合补充

+### 01-01 安装、路由分组与参数

#### 安装并启动 Gin

```bash
go get github.com/gin-gonic/gin
```

```go
package main

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default() // 含 Logger 与 Recovery
    r.GET("/healthz", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"status": "ok"})
    })
    _ = r.Run(":8080")
}
```

`gin.Default()` 适合起步。需要精确控制中间件时使用 `gin.New()`，并自行添加 `gin.Logger()`、`gin.Recovery()` 等中间件；生产服务不能遗漏 Recovery。

#### 分组和版本

```go
v1 := r.Group("/api/v1")
{
    v1.GET("/users/:id", getUser)
    v1.GET("/users", listUsers)
    v1.POST("/users", createUser)
}
```

版本前缀让未来的破坏性修改可以通过 `/api/v2` 演进，而不立刻影响已有客户端。路由注册应集中管理，避免在多个包中隐式注册，导致难以查找冲突。

#### 四种常见输入

```go
func inspect(c *gin.Context) {
    id := c.Param("id")                  // /users/:id
    page, exists := c.GetQuery("page")   // ?page=1，保留“是否传入”信息
    agent := c.GetHeader("User-Agent")   // Header
    _ = id; _ = page; _ = exists; _ = agent
}
```

JSON body 在下一节和第 2 阶段通过 `ShouldBindJSON` 绑定。不要用路径、查询参数或 Header 传递密码等敏感值；它们更容易出现在日志、历史记录和代理中。

#### 404 与 405

`404` 通常表示路径或分组前缀不匹配；`405` 表示路径存在但方法不匹配。排查时使用精确的 `curl -i -X METHOD URL`，不要只在浏览器地址栏中测试，因为地址栏只会发 GET。

#### 本节验收

- 运行 `/healthz` 和 `/api/v1/users/:id`。
- 能说明 `Param`、`Query`、`GetQuery`、`GetHeader` 的使用边界。
- 用错误方法请求一个已注册路径，观察 404/405 的差异。

+### 01-02 内存用户 CRUD 实践

本节先完成一个单文件版本的用户 API。它不是最终项目结构，而是验证路由、JSON 和并发安全的最短路径；第 4 阶段会将它拆成分层项目。

```go
package main

import (
    "net/http"
    "strconv"
    "sync"
    "github.com/gin-gonic/gin"
)

type User struct {
    ID uint64 `json:"id"`
    Name string `json:"name"`
    Email string `json:"email"`
}

type CreateUserInput struct {
    Name string `json:"name" binding:"required,min=2,max=50"`
    Email string `json:"email" binding:"required,email"`
}

var (
    users = map[uint64]User{}
    nextID uint64 = 1
    mu sync.RWMutex
)

func main() {
    r := gin.Default()
    v1 := r.Group("/api/v1")
    v1.POST("/users", createUser)
    v1.GET("/users", listUsers)
    v1.GET("/users/:id", getUser)
    _ = r.Run(":8080")
}

func createUser(c *gin.Context) {
    var in CreateUserInput
    if err := c.ShouldBindJSON(&in); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"message": "invalid request"})
        return
    }
    mu.Lock()
    user := User{ID: nextID, Name: in.Name, Email: in.Email}
    users[nextID] = user
    nextID++
    mu.Unlock()
    c.JSON(http.StatusCreated, user)
}

func listUsers(c *gin.Context) {
    mu.RLock()
    result := make([]User, 0, len(users))
    for _, user := range users { result = append(result, user) }
    mu.RUnlock()
    c.JSON(http.StatusOK, result)
}

func getUser(c *gin.Context) {
    id, err := strconv.ParseUint(c.Param("id"), 10, 64)
    if err != nil { c.JSON(http.StatusBadRequest, gin.H{"message": "invalid id"}); return }
    mu.RLock(); user, ok := users[id]; mu.RUnlock()
    if !ok { c.JSON(http.StatusNotFound, gin.H{"message": "user not found"}); return }
    c.JSON(http.StatusOK, user)
}
```

验证：

```bash
curl -i -X POST http://localhost:8080/api/v1/users -H "Content-Type: application/json" -d '{"name":"Ada","email":"ada@example.com"}'
curl -i http://localhost:8080/api/v1/users
curl -i http://localhost:8080/api/v1/users/1
```

补充 `PATCH /users/:id` 与 `DELETE /users/:id` 时，先解析并验证 ID，再确认资源存在。共享 map 需要锁；实际服务中的数据不会放在全局 map，进程重启后数据消失也正是下一阶段接入数据库的动机。

**验收：**能创建、列表、详情、修改和删除；无效 ID 返回 400，不存在 ID 返回 404，缺失 `Content-Type` 或字段时返回 400。

