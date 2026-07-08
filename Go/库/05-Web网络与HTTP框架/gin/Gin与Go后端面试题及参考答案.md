# Gin 与 Go 后端面试题及参考答案

这份文档用于复习 Go 后端工程师面试，重点围绕：

- Go 语言基础
- HTTP 与 RESTful API
- Gin 框架
- 参数绑定与统一响应
- 中间件、JWT、权限控制
- 项目分层与工程化
- GORM 与数据库
- Redis、缓存、限流
- 测试、Swagger、日志
- Docker、部署、生产排错
- 任务管理系统项目追问
- 综合系统设计题

建议使用方式：

```text
第一遍：先看问题，自己口头回答。
第二遍：对照参考答案补缺口。
第三遍：把答案改成自己的表达。
第四遍：结合项目代码讲出真实实现。
```

面试时不要背书式回答。更好的方式是：

```text
先给结论。
再解释原因。
再结合项目经验。
最后补充边界和风险。
```

---

## 一、Go 语言基础

### 1. Go 语言有什么特点？

参考答案：

Go 是一门偏工程化的静态类型编译型语言，常用于后端服务、云原生、基础设施和微服务开发。

主要特点：

- 编译速度快。
- 部署简单，通常编译成一个二进制文件。
- 原生支持并发，使用 goroutine 和 channel。
- 标准库强大，尤其是 `net/http`。
- 语法简洁，团队协作成本低。
- 自带垃圾回收。
- 工具链完善，比如 `go test`、`go fmt`、`go mod`。

可以结合后端项目回答：

> 我学习 Gin 时感受到 Go 很适合写后端 API。比如一个 Gin 服务可以直接编译成二进制，再配合 Docker 部署，不需要像一些语言那样准备复杂运行环境。

---

### 2. Go 是值传递还是引用传递？

参考答案：

Go 中函数参数传递本质上都是值传递。

但是有些类型的值本身包含指向底层数据的引用，比如：

- slice
- map
- channel
- function
- interface
- pointer

例如：

```go
func update(m map[string]int) {
    m["a"] = 1
}
```

这里 map 作为参数传递时，map header 被复制了，但它仍然指向同一个底层哈希表，所以函数内部修改会影响外部。

而普通 struct：

```go
type User struct {
    Name string
}

func update(u User) {
    u.Name = "new"
}
```

不会影响外部，因为 struct 被复制了一份。

如果想修改外部 struct，需要传指针：

```go
func update(u *User) {
    u.Name = "new"
}
```

---

### 3. slice 和 array 有什么区别？

参考答案：

array 是固定长度数组，长度是类型的一部分。

```go
var a [3]int
var b [4]int
```

`[3]int` 和 `[4]int` 是不同类型。

slice 是动态视图，底层引用数组，包含：

```text
指向底层数组的指针
长度 len
容量 cap
```

slice 更常用：

```go
nums := []int{1, 2, 3}
nums = append(nums, 4)
```

面试补充点：

- append 可能触发扩容。
- 扩容后底层数组可能变化。
- 多个 slice 可能共享同一个底层数组。
- 大 slice 截取小 slice 时，可能导致底层大数组无法释放。

---

### 4. map 是否并发安全？

参考答案：

Go 原生 map 不是并发安全的。

多个 goroutine 同时读写 map 可能 panic：

```text
fatal error: concurrent map writes
```

解决方式：

1. 使用 `sync.Mutex` 或 `sync.RWMutex`。
2. 使用 `sync.Map`。
3. 通过 channel 串行化访问。

示例：

```go
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}

func (s *SafeMap) Set(key string, value int) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.m[key] = value
}

func (s *SafeMap) Get(key string) (int, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    v, ok := s.m[key]
    return v, ok
}
```

项目关联：

> 第2阶段内存版用户 CRUD 使用 map 保存用户，这只适合学习。真实项目要用数据库，或者至少要考虑并发安全。

---

### 5. goroutine 是什么？和线程有什么区别？

参考答案：

goroutine 是 Go 的轻量级并发执行单元，由 Go runtime 调度。

和系统线程相比：

- goroutine 初始栈更小。
- 创建成本更低。
- Go runtime 使用 M:N 调度，把多个 goroutine 调度到多个 OS 线程上。
- 开发者使用 `go func()` 就能启动。

示例：

```go
go func() {
    fmt.Println("run in goroutine")
}()
```

注意：

- goroutine 不是越多越好。
- goroutine 泄漏会导致内存和资源问题。
- 长时间阻塞的 goroutine 要考虑 context 取消。

---

### 6. channel 是什么？适合什么场景？

参考答案：

channel 是 goroutine 之间通信的管道。

创建：

```go
ch := make(chan int)
```

发送：

```go
ch <- 1
```

接收：

```go
v := <-ch
```

适合：

- goroutine 间传递数据。
- 控制并发。
- 通知任务完成。
- 实现生产者消费者模型。

不适合：

- 所有共享状态都强行用 channel。
- 简单共享计数器可以用 mutex 或 atomic。

面试可以说：

> Go 提倡通过通信共享内存，但不是说所有场景都必须使用 channel。实际项目中我会根据场景选择 mutex、channel 或其他同步方式。

---

### 7. defer 的执行顺序是什么？

参考答案：

`defer` 在函数返回前执行，多个 defer 按后进先出顺序执行。

示例：

```go
func demo() {
    defer fmt.Println("first")
    defer fmt.Println("second")
    defer fmt.Println("third")
}
```

输出：

```text
third
second
first
```

常见用途：

- 关闭文件。
- 释放锁。
- 关闭数据库 rows。
- recover panic。

注意：

```go
defer mu.Unlock()
```

应该在加锁成功后立即写，避免中间 return 忘记释放锁。

---

### 8. panic 和 error 有什么区别？

参考答案：

`error` 是 Go 中正常的错误处理方式，适合可预期错误。

例如：

- 参数错误。
- 用户不存在。
- 数据库查询失败。
- 文件不存在。

`panic` 适合不可恢复的程序错误。

例如：

- 数组越界。
- nil 指针。
- 初始化阶段关键依赖失败。

后端服务中不应该用 panic 处理普通业务错误。

Gin 项目中：

- handler/service 返回 error。
- middleware 中使用 Recovery 捕获 panic。
- 对外返回统一 `500`。
- 详细 panic 写日志。

---

### 9. context.Context 有什么作用？

参考答案：

`context.Context` 用于在请求链路中传递：

- 取消信号。
- 超时时间。
- 请求范围内的值。

后端项目中常见用法：

```go
ctx := c.Request.Context()
user, err := service.GetUser(ctx, userID)
```

repository 中：

```go
db.WithContext(ctx).First(&user, id)
```

Redis 中：

```go
rdb.Get(ctx, key).Result()
```

意义：

- 客户端断开后，后续数据库/Redis 操作可以尽快停止。
- 防止慢请求长期占用资源。
- 服务关闭时可以优雅取消任务。

注意：

- 不要在请求链路中随便用 `context.Background()`。
- `context.Value` 不适合传大量业务参数。
- context 应该作为函数第一个参数。

---

### 10. interface 的作用是什么？

参考答案：

interface 定义行为，不定义具体实现。

例如 service 依赖 repository：

```go
type UserRepository interface {
    FindByEmail(ctx context.Context, email string) (*model.User, error)
    Create(ctx context.Context, user *model.User) error
}

type UserService struct {
    repo UserRepository
}
```

好处：

- 降低耦合。
- 方便测试替换 fake repository。
- 让 service 不关心底层是 MySQL、PostgreSQL 还是内存实现。

面试补充：

> Go 的接口是隐式实现的。一个类型只要实现了接口要求的方法，就自动满足接口，不需要显式声明 implements。

---

## 二、HTTP 与 RESTful API

### 11. HTTP 请求由哪些部分组成？

参考答案：

HTTP 请求主要包含：

- 请求方法 Method：GET、POST、PUT、DELETE 等。
- URL/Path：请求资源路径。
- Query 参数：例如 `?page=1`。
- Header：请求元信息，如 Authorization、Content-Type。
- Body：请求体，常见 JSON。

示例：

```http
POST /api/v1/users?page=1 HTTP/1.1
Host: localhost:8080
Content-Type: application/json
Authorization: Bearer token

{
  "email": "a@example.com"
}
```

Gin 中对应：

```go
c.Param("id")
c.Query("page")
c.GetHeader("Authorization")
c.ShouldBindJSON(&req)
```

---

### 12. GET 和 POST 有什么区别？

参考答案：

GET 通常用于查询资源，参数一般放在 URL query 中，语义上应该是安全、幂等的。

POST 通常用于创建资源或提交操作，请求数据通常放在 body 中。

区别：

- GET 参数在 URL 中，POST 参数通常在 body。
- GET 适合查询，POST 适合创建。
- GET 一般可以被缓存，POST 通常不缓存。
- GET 不应该改变服务端状态。

注意：

> 实际上 HTTP 方法是语义约定，不是物理限制。GET 也能带 body，但不推荐；POST 也能查询，但 RESTful 设计中不建议滥用。

---

### 13. PUT 和 PATCH 有什么区别？

参考答案：

PUT 通常表示完整更新资源。

PATCH 通常表示部分更新资源。

例如：

```http
PUT /api/v1/tasks/1
```

请求体可能包含完整任务字段。

```http
PATCH /api/v1/tasks/1/status
```

只更新状态字段。

项目中可以这样设计：

- `PUT /tasks/:id` 更新标题、描述、截止时间。
- `PATCH /tasks/:id/status` 单独更新状态。

---

### 14. 常见 HTTP 状态码有哪些？

参考答案：

常见状态码：

```text
200 OK：请求成功。
201 Created：创建成功。
204 No Content：成功但无响应体。
400 Bad Request：参数错误。
401 Unauthorized：未登录或 token 无效。
403 Forbidden：无权限。
404 Not Found：资源不存在。
405 Method Not Allowed：方法不允许。
409 Conflict：资源冲突。
429 Too Many Requests：请求过于频繁。
500 Internal Server Error：服务器内部错误。
```

项目中：

- 注册成功返回 201。
- 参数校验失败返回 400。
- 未带 token 返回 401。
- 普通用户访问管理员接口返回 403。
- 任务不存在返回 404。
- 邮箱重复返回 409。
- 登录失败过多返回 429。

---

### 15. 什么是 RESTful API？

参考答案：

RESTful API 是一种面向资源的 API 设计风格。

核心思想：

- URL 表示资源。
- HTTP 方法表示动作。
- 状态码表示结果。

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
GET  /getUsers
POST /createUser
POST /deleteUser
```

面试回答可以结合项目：

> 在任务管理系统中，我用 `/api/v1/tasks` 表示任务资源，用 GET 查询、POST 创建、PUT 更新、DELETE 删除，这样接口语义更清楚，也方便前后端协作。

---

## 三、Gin 框架基础

### 16. Gin 是什么？为什么用 Gin？

参考答案：

Gin 是 Go 生态中常用的 Web 框架，主要用于构建 HTTP API。

优点：

- 路由性能好。
- API 简洁。
- 支持中间件。
- 支持参数绑定和校验。
- 社区成熟。
- 适合快速开发 RESTful API。

相比标准库：

- 标准库能写 HTTP 服务，但路由分组、中间件、参数绑定都要自己处理。
- Gin 提供了更完整的 Web 开发能力。

---

### 17. `gin.Default()` 和 `gin.New()` 有什么区别？

参考答案：

`gin.Default()` 默认包含：

- Logger 中间件。
- Recovery 中间件。

```go
r := gin.Default()
```

`gin.New()` 创建一个没有默认中间件的引擎：

```go
r := gin.New()
r.Use(gin.Logger())
r.Use(gin.Recovery())
```

项目中：

- 学习阶段可以用 `gin.Default()`。
- 工程化项目更常用 `gin.New()`，然后注册自定义 logger、recovery、request_id 等中间件。

---

### 18. Gin 如何获取路径参数、查询参数和 Header？

参考答案：

路径参数：

```go
id := c.Param("id")
```

查询参数：

```go
page := c.DefaultQuery("page", "1")
keyword := c.Query("keyword")
```

Header：

```go
token := c.GetHeader("Authorization")
```

示例：

```http
GET /api/v1/users/10?page=1
Authorization: Bearer token
```

对应：

```go
c.Param("id")              // 10
c.Query("page")            // 1
c.GetHeader("Authorization")
```

---

### 19. Gin 如何绑定 JSON 请求体？

参考答案：

定义请求结构体：

```go
type CreateUserRequest struct {
    Username string `json:"username" binding:"required"`
    Email    string `json:"email" binding:"required,email"`
}
```

handler 中：

```go
var req CreateUserRequest
if err := c.ShouldBindJSON(&req); err != nil {
    response.Fail(c, http.StatusBadRequest, response.CodeInvalidRequest, "invalid request")
    return
}
```

注意：

- 请求头应为 `Content-Type: application/json`。
- JSON 字段名要和 tag 对应。
- binding 校验失败也会返回 error。

---

### 20. `BindJSON` 和 `ShouldBindJSON` 有什么区别？

参考答案：

`BindJSON` 绑定失败时会自动写入 400 响应，并且可能影响后续统一响应控制。

`ShouldBindJSON` 只返回 error，不自动写响应。

项目中推荐使用：

```go
if err := c.ShouldBindJSON(&req); err != nil {
    response.FailWithData(...)
    return
}
```

原因：

- 方便统一响应格式。
- 方便格式化字段级错误。
- 不让框架替你决定响应。

---

## 四、参数校验与统一响应

### 21. Gin binding 常用规则有哪些？

参考答案：

常用规则：

```text
required：必填。
min：最小长度或最小值。
max：最大长度或最大值。
email：邮箱格式。
gte：大于等于。
lte：小于等于。
oneof：枚举。
omitempty：为空时跳过后续校验。
```

示例：

```go
type CreateTaskRequest struct {
    Title       string `json:"title" binding:"required,min=1,max=100"`
    Description string `json:"description" binding:"omitempty,max=1000"`
    Status      string `json:"status" binding:"omitempty,oneof=todo doing done cancelled"`
}
```

---

### 22. 为什么要做统一响应？

参考答案：

统一响应可以让前端和调用方稳定处理结果。

如果每个接口返回不同结构，调用方要写大量特殊判断。

推荐结构：

```go
type Body struct {
    Code    int         `json:"code"`
    Message string      `json:"message"`
    Data    interface{} `json:"data,omitempty"`
}
```

成功：

```json
{
  "code": 0,
  "message": "ok",
  "data": {}
}
```

失败：

```json
{
  "code": 40001,
  "message": "invalid request"
}
```

分页：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "items": [],
    "total": 0,
    "page": 1,
    "page_size": 10
  }
}
```

---

### 23. HTTP 状态码和业务错误码有什么区别？

参考答案：

HTTP 状态码表达协议层结果，比如：

- 400 参数错误。
- 401 未登录。
- 404 不存在。
- 500 服务错误。

业务错误码表达业务层原因，比如：

- 40001 参数错误。
- 40101 未登录。
- 40901 邮箱重复。

两者都需要。

只用 HTTP 状态码不够细。

只用业务 code 而 HTTP 全部返回 200，不利于网关、监控、日志和通用客户端处理。

---

### 24. 你如何处理参数校验错误？

参考答案：

我会在 handler 层使用 `ShouldBindJSON` 或 `ShouldBindQuery`，如果校验失败，统一返回 400。

如果需要字段级错误，可以把 validator 错误格式化：

```go
var validationErrors validator.ValidationErrors
if errors.As(err, &validationErrors) {
    for _, fieldError := range validationErrors {
        result[fieldError.Field()] = fieldError.Tag()
    }
}
```

最终响应：

```json
{
  "code": 40001,
  "message": "invalid request",
  "data": {
    "email": "email",
    "password": "min"
  }
}
```

项目中这样做的好处是前端能明确知道哪个字段错了。

---

### 25. 404 和 405 如何统一返回 JSON？

参考答案：

Gin 可以设置：

```go
r.NoRoute(func(c *gin.Context) {
    response.Fail(c, http.StatusNotFound, response.CodeNotFound, "route not found")
})
```

405 要先开启：

```go
r.HandleMethodNotAllowed = true
```

再设置：

```go
r.NoMethod(func(c *gin.Context) {
    response.Fail(c, http.StatusMethodNotAllowed, response.CodeInvalidRequest, "method not allowed")
})
```

这样框架级错误也能保持统一响应格式。

---

## 五、中间件、JWT 与权限

### 26. Gin 中间件的执行流程是什么？

参考答案：

Gin 中间件是洋葱模型。

示例：

```go
func Middleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        // handler 前
        c.Next()
        // handler 后
    }
}
```

多个中间件按注册顺序进入，按相反顺序退出。

常见用途：

- 日志。
- Recovery。
- CORS。
- RequestID。
- JWT 认证。
- 限流。

---

### 27. `c.Next()` 和 `c.Abort()` 有什么区别？

参考答案：

`c.Next()` 表示继续执行后续中间件和 handler。

`c.Abort()` 表示中断后续 handler 链。

认证失败时：

```go
if token == "" {
    response.Fail(c, http.StatusUnauthorized, response.CodeUnauthorized, "unauthorized")
    c.Abort()
    return
}
```

注意：

`c.Abort()` 后最好立即 `return`，否则当前函数后续代码仍会执行。

---

### 28. JWT 是什么？

参考答案：

JWT 是 JSON Web Token，用于在客户端和服务端之间传递可验证的身份信息。

结构：

```text
Header.Payload.Signature
```

服务端登录成功后生成 token，客户端后续请求携带：

```http
Authorization: Bearer token
```

服务端验证 token 签名和过期时间，解析出用户 ID、角色等信息。

优点：

- 无状态。
- 适合 API 认证。
- 不需要每次查 session。

缺点：

- 签发后在过期前不容易主动失效。
- token 泄露会有风险。
- payload 只是编码，不是加密，不能放敏感信息。

---

### 29. JWT 中应该存什么？不应该存什么？

参考答案：

可以存：

- user_id
- role
- expire time
- issued at

不应该存：

- 密码。
- 密码哈希。
- 手机号、身份证等敏感信息。
- 大量业务数据。

原因：

JWT payload 可以被客户端解码看到，只是不能被篡改。

---

### 30. 401 和 403 有什么区别？

参考答案：

401 表示未认证：

```text
没有 token。
token 格式错误。
token 过期。
token 签名无效。
```

403 表示已认证但无权限：

```text
普通用户访问管理员接口。
用户访问不属于自己的资源。
```

项目中：

- 不带 token 访问 profile 返回 401。
- 普通用户访问 admin 返回 403。
- 用户访问别人任务，可以返回 403，也可以返回 404 隐藏资源存在性。

---

### 31. 如何实现 JWT 认证中间件？

参考答案：

核心步骤：

1. 从 Header 读取 Authorization。
2. 检查 Bearer 格式。
3. 解析 token。
4. 验证签名和过期时间。
5. 把 user_id、role 写入 Gin Context。
6. 放行。

示例：

```go
func Auth(jwtManager *JWTManager) gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" {
            response.Fail(c, http.StatusUnauthorized, response.CodeUnauthorized, "unauthorized")
            c.Abort()
            return
        }

        parts := strings.SplitN(authHeader, " ", 2)
        if len(parts) != 2 || parts[0] != "Bearer" {
            response.Fail(c, http.StatusUnauthorized, response.CodeUnauthorized, "unauthorized")
            c.Abort()
            return
        }

        claims, err := jwtManager.Parse(parts[1])
        if err != nil {
            response.Fail(c, http.StatusUnauthorized, response.CodeUnauthorized, "unauthorized")
            c.Abort()
            return
        }

        c.Set("user_id", claims.UserID)
        c.Set("role", claims.Role)
        c.Next()
    }
}
```

---

### 32. CORS 是什么？如何处理？

参考答案：

CORS 是浏览器的跨域资源共享机制。当前端页面和后端 API 不同源时，浏览器会检查后端是否允许跨域。

常见响应头：

```text
Access-Control-Allow-Origin
Access-Control-Allow-Methods
Access-Control-Allow-Headers
```

如果请求带 Authorization，必须允许：

```text
Access-Control-Allow-Headers: Content-Type,Authorization
```

OPTIONS 预检请求要直接返回：

```go
if c.Request.Method == http.MethodOptions {
    c.AbortWithStatus(http.StatusNoContent)
    return
}
```

生产环境不要无脑允许所有 Origin。

---

## 六、项目分层与工程化

### 33. 你如何设计 Gin 项目目录？

参考答案：

常见结构：

```text
cmd/server/main.go
internal/config
internal/database
internal/model
internal/repository
internal/service
internal/handler
internal/middleware
internal/router
internal/response
pkg/logger
```

职责：

- main：加载配置、初始化依赖、启动服务。
- router：注册路由。
- handler：处理 HTTP 输入输出。
- service：处理业务逻辑。
- repository：处理数据访问。
- model：数据库模型。
- middleware：认证、日志、CORS 等。
- response：统一响应。

---

### 34. handler、service、repository 分别负责什么？

参考答案：

handler：

- 读取路径参数、查询参数、Header、JSON body。
- 调用 service。
- 把 service 结果转换为 HTTP 响应。

service：

- 处理业务规则。
- 组合多个 repository。
- 判断业务错误。
- 不依赖 Gin。

repository：

- 操作数据库。
- 封装 GORM 查询。
- 把底层错误转换为 repository 错误。

例子：

> 创建任务时，handler 负责绑定请求；service 负责设置默认状态、校验任务归属；repository 负责写入 tasks 表。

---

### 35. 为什么 service 不应该依赖 `*gin.Context`？

参考答案：

因为 service 是业务层，不应该和 HTTP 框架绑定。

如果 service 接收 `*gin.Context`：

- 测试困难。
- 无法复用于 gRPC、命令行、消息消费者。
- 业务层和 Web 框架耦合。

推荐：

```go
func (s *TaskService) Create(ctx context.Context, userID uint, req CreateTaskInput) (*TaskDTO, error)
```

handler 中：

```go
task, err := h.service.Create(c.Request.Context(), userID, input)
```

---

### 36. 什么是依赖注入？

参考答案：

依赖注入就是对象需要什么依赖，不在内部偷偷创建，而是由外部传入。

推荐：

```go
repo := repository.NewUserRepository(db)
svc := service.NewUserService(repo)
handler := handler.NewUserHandler(svc)
```

不推荐：

```go
func NewUserHandler() *UserHandler {
    db := database.New()
    repo := repository.NewUserRepository(db)
    svc := service.NewUserService(repo)
    return &UserHandler{svc: svc}
}
```

好处：

- 依赖关系清晰。
- 方便测试替换。
- 生命周期集中管理。

---

### 37. DTO 和 Model 有什么区别？

参考答案：

Model 面向数据库结构。

DTO 面向接口输入输出。

数据库模型可能包含：

- password_hash
- deleted_at
- internal_status

这些不应该直接返回给前端。

示例：

```go
type User struct {
    ID           uint
    Email        string
    PasswordHash string
}

type UserDTO struct {
    ID    uint   `json:"id"`
    Email string `json:"email"`
}
```

好处：

- 防止敏感字段泄露。
- 接口结构不被数据库结构绑死。
- 请求字段和响应字段可以分别控制。

---

## 七、GORM 与数据库

### 38. GORM 是什么？

参考答案：

GORM 是 Go 生态常用 ORM 框架，用于简化数据库操作。

它可以：

- 定义模型。
- CRUD。
- 自动迁移。
- 事务。
- 关联查询。
- 软删除。
- hooks。

示例：

```go
db.Create(&user)
db.First(&user, id)
db.Where("email = ?", email).Take(&user)
db.Model(&user).Updates(...)
db.Delete(&user)
```

注意：

GORM 不是替代 SQL 基础。复杂查询、性能优化、索引设计仍然要理解数据库。

---

### 39. 如何初始化 GORM 连接？

参考答案：

示例：

```go
db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
if err != nil {
    return nil, err
}

sqlDB, err := db.DB()
if err != nil {
    return nil, err
}

sqlDB.SetMaxIdleConns(10)
sqlDB.SetMaxOpenConns(100)
sqlDB.SetConnMaxLifetime(time.Hour)
```

要点：

- DSN 放配置，不写死。
- 设置连接池。
- 启动时 Ping。
- 生产中谨慎使用 AutoMigrate。

---

### 40. GORM 的 `First`、`Take`、`Find` 有什么区别？

参考答案：

`First`：

- 查询第一条。
- 默认按主键排序。
- 查不到返回 `gorm.ErrRecordNotFound`。

`Take`：

- 查询一条。
- 不指定排序。
- 查不到返回 `gorm.ErrRecordNotFound`。

`Find`：

- 查询多条。
- 查不到通常返回空切片，不是 ErrRecordNotFound。

列表接口中空列表不是错误，应该返回：

```json
{
  "items": [],
  "total": 0
}
```

---

### 41. GORM 软删除是什么？

参考答案：

如果模型中包含 `gorm.DeletedAt`，调用 `Delete` 时通常不会物理删除，而是设置 `deleted_at`。

```go
type Task struct {
    gorm.Model
    Title string
}
```

删除：

```go
db.Delete(&task)
```

普通查询会自动加条件：

```sql
deleted_at IS NULL
```

如果要查软删除数据：

```go
db.Unscoped().First(&task, id)
```

项目中任务删除适合软删除，方便恢复和审计。

---

### 42. 如何处理唯一索引冲突？

参考答案：

数据库层必须加唯一索引，比如 email。

```go
Email string `gorm:"uniqueIndex"`
```

service 可以先查是否存在，给友好提示。

但并发情况下，仍然可能两个请求同时通过检查，所以数据库唯一索引要兜底。

repository 中捕获重复键错误，转换为业务错误：

```go
if isDuplicateKey(err) {
    return ErrEmailExists
}
```

handler 返回：

```text
HTTP 409
code 40901
```

---

### 43. 什么情况下需要事务？

参考答案：

当一个业务操作包含多个数据库写操作，并且它们必须同时成功或同时失败时，需要事务。

例如：

- 创建订单 + 扣库存。
- 转账扣款 + 入账。
- 创建用户 + 创建用户资料。

GORM：

```go
err := db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&order).Error; err != nil {
        return err
    }
    if err := tx.Model(&stock).Update("count", gorm.Expr("count - ?", n)).Error; err != nil {
        return err
    }
    return nil
})
```

注意：

- 事务中要使用 `tx`，不是外层 `db`。
- 每一步错误都要返回。
- 不要吞掉错误。

---

### 44. 如何做分页查询？

参考答案：

分页参数：

```text
page 从 1 开始
page_size 设置最大值
offset = (page - 1) * page_size
```

GORM：

```go
query := db.Model(&Task{}).Where("user_id = ?", userID)

if status != "" {
    query = query.Where("status = ?", status)
}

if err := query.Count(&total).Error; err != nil {
    return nil, 0, err
}

if err := query.Order("id DESC").Offset(offset).Limit(pageSize).Find(&tasks).Error; err != nil {
    return nil, 0, err
}
```

注意：

- Count 和 Find 要使用同一组筛选条件。
- page_size 要限制最大值。
- 大数据深分页可能需要游标分页优化。

---

### 45. 数据库索引有什么作用？

参考答案：

索引用于加速查询，但会增加写入成本和存储空间。

适合建索引的字段：

- 经常用于 WHERE 的字段。
- 经常用于 JOIN 的字段。
- 经常用于 ORDER BY 的字段。
- 唯一约束字段。

任务系统中：

- `users.email` 唯一索引。
- `tasks.user_id` 索引。
- `tasks.status` 索引。
- 可能需要 `(user_id, status, created_at)` 联合索引。

面试补充：

> 索引不是越多越好。要根据查询模式设计，并通过 EXPLAIN 验证。

---

## 八、Redis、缓存与限流

### 46. Redis 在后端项目中常见用途有哪些？

参考答案：

常见用途：

- 缓存。
- 分布式锁。
- 计数器。
- 限流。
- 验证码。
- 排行榜。
- 会话存储。
- 消息队列或 Stream。

在任务管理系统中：

- 缓存用户 profile。
- 登录失败次数限制。
- 任务详情缓存。

---

### 47. 什么是缓存穿透、击穿、雪崩？

参考答案：

缓存穿透：

请求的数据缓存没有，数据库也没有，导致每次都打到数据库。

解决：

- 参数校验。
- 缓存空值。
- 布隆过滤器。

缓存击穿：

热点 key 过期，大量请求同时打到数据库。

解决：

- 热点 key 延长 TTL。
- 互斥锁。
- 后台刷新。

缓存雪崩：

大量 key 同时过期，数据库瞬间压力大。

解决：

- TTL 加随机值。
- 分批预热。
- 限流降级。

---

### 48. Cache Aside 模式是什么？

参考答案：

读取：

```text
先读缓存。
命中则返回。
未命中查数据库。
查到后写缓存。
返回结果。
```

更新：

```text
先更新数据库。
再删除缓存。
```

为什么通常删除缓存，而不是直接更新缓存？

因为数据库更新和缓存更新不是原子操作，直接更新更容易出现不一致。删除后下次读取会重新加载。

---

### 49. Redis key 如何设计？

参考答案：

key 应该：

- 有业务前缀。
- 可读。
- 包含资源类型。
- 包含唯一标识。
- 设置 TTL。

示例：

```text
task-api:user:profile:1
task-api:task:detail:1:100
task-api:login:fail:a@example.com
```

不推荐：

```text
u1
data
cache
```

---

### 50. 如何实现登录失败限制？

参考答案：

规则：

```text
同一邮箱 15 分钟内失败超过 5 次，返回 429。
登录成功后清除计数。
```

Redis：

```go
key := "task-api:login:fail:" + email
count, err := rdb.Incr(ctx, key).Result()
if count == 1 {
    rdb.Expire(ctx, key, 15*time.Minute)
}
```

检查：

```go
count, err := rdb.Get(ctx, key).Int()
if count >= 5 {
    return ErrTooManyLoginAttempts
}
```

返回：

```text
HTTP 429
code 42901
```

---

### 51. Redis 挂了接口是否应该失败？

参考答案：

要看 Redis 在当前场景中的角色。

如果 Redis 是缓存：

- 可以降级查数据库。
- 记录 warn 日志。
- 不一定让接口失败。

如果 Redis 用于安全限制：

- 要根据安全策略决定。
- 学习项目可以降级。
- 高安全系统可能拒绝登录。

面试回答：

> 我不会简单地所有 Redis 错误都返回 500，而是区分缓存场景和安全场景。缓存失败通常可以降级，限流失败要看业务安全要求。

---

## 九、测试、日志与 Swagger

### 52. handler 测试应该测什么？

参考答案：

handler 测试关注 HTTP 行为：

- 路由是否正确。
- 状态码是否正确。
- 请求参数错误是否返回 400。
- 未登录是否返回 401。
- 响应 JSON 是否符合预期。

使用：

```go
req := httptest.NewRequest(http.MethodGet, "/ping", nil)
w := httptest.NewRecorder()
r.ServeHTTP(w, req)
```

handler 测试不应该深度依赖真实数据库，可以使用 fake service。

---

### 53. service 测试应该测什么？

参考答案：

service 测试关注业务规则：

- 注册成功。
- 邮箱重复。
- 密码错误。
- 用户不能访问别人任务。
- 非法状态失败。

通常使用 fake repository。

推荐表格驱动测试：

```go
tests := []struct {
    name    string
    input   Input
    wantErr bool
}{...}
```

---

### 54. 为什么要用 zap 日志？

参考答案：

zap 是高性能结构化日志库。

结构化日志比普通字符串更适合搜索和分析。

示例：

```go
logger.Info("request completed",
    zap.String("request_id", requestID),
    zap.String("method", method),
    zap.String("path", path),
    zap.Int("status", status),
    zap.Duration("latency", latency),
)
```

生产中要注意：

- 不记录明文密码。
- 不记录完整 token。
- 错误日志要带上下文。
- request_id 有助于排查链路。

---

### 55. Swagger 的作用是什么？

参考答案：

Swagger/OpenAPI 用于描述 API 文档。

它能说明：

- 接口路径。
- 请求方法。
- 请求参数。
- 请求体。
- 响应结构。
- 认证要求。

Gin 项目中可以用 `swaggo`：

```bash
swag init -g cmd/server/main.go -o docs
```

注意：

Swagger 不是测试。它说明接口契约，但不能证明接口行为正确。

---

## 十、Docker、部署与生产排错

### 56. Go 项目如何写 Dockerfile？

参考答案：

推荐多阶段构建：

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o /app/server ./cmd/server

FROM alpine:3.20
WORKDIR /app
COPY --from=builder /app/server /app/server
COPY config /app/config
EXPOSE 8080
CMD ["/app/server", "-config=/app/config/config.yaml"]
```

好处：

- 运行镜像更小。
- 不包含完整 Go 编译环境。
- 构建和运行环境分离。

---

### 57. Docker Compose 中为什么不能用 `127.0.0.1` 连接 MySQL？

参考答案：

在容器中，`127.0.0.1` 指的是当前容器自己，不是宿主机，也不是 MySQL 容器。

Compose 中服务之间应该用服务名访问：

```text
mysql:3306
redis:6379
```

例如 app 连接 MySQL：

```text
root:password@tcp(mysql:3306)/task_api?parseTime=True&loc=Local
```

这是部署面试中很常见的问题。

---

### 58. `/healthz` 和 `/readyz` 有什么区别？

参考答案：

`/healthz` 表示进程是否活着。

它通常只返回：

```json
{"status":"ok"}
```

`/readyz` 表示服务是否准备好接收流量，需要检查依赖：

- 数据库。
- Redis。
- 消息队列。

如果 Redis 或数据库不可用，`/readyz` 可以返回 503。

---

### 59. Nginx 502 如何排查？

参考答案：

502 表示 Nginx 无法从上游服务拿到正常响应。

排查顺序：

```bash
docker compose ps
docker compose logs nginx
docker compose logs app
```

检查：

- app 是否启动。
- app 是否监听 `:8080`。
- Nginx upstream 是否写 `app:8080`。
- app 是否启动后崩溃。
- Docker 网络是否正常。

---

### 60. 生产环境配置应该如何管理？

参考答案：

原则：

- 配置和代码分离。
- 敏感信息不提交 Git。
- `.env.example` 提供示例。
- 真实 `.env` 不提交。
- 生产密钥通过环境变量或密钥管理服务注入。

`.gitignore`：

```gitignore
.env
.env.*
!.env.example
```

配置项：

- 数据库 DSN。
- Redis 地址。
- JWT Secret。
- 日志级别。
- Gin mode。

---

## 十一、任务管理系统项目追问

### 61. 请介绍一下你的任务管理系统项目。

参考答案：

可以这样回答：

> 这个项目是一个基于 Gin 的任务管理系统后端 API，主要功能包括用户注册、登录、JWT 认证、获取个人信息、任务创建、任务列表、任务详情、任务更新、任务状态修改和删除。技术上使用 Gin 做 HTTP API，GORM 操作数据库，Redis 做登录失败限制和缓存，JWT 做认证，zap 做日志，Swagger 生成接口文档，Docker Compose 做本地部署。

继续补充架构：

> 项目采用 handler、service、repository 分层。handler 处理 HTTP 参数和响应，service 处理业务逻辑，repository 负责数据库访问。这样接入 GORM、Redis 或后续测试替身时，代码边界比较清楚。

---

### 62. 任务系统如何保证用户只能访问自己的任务？

参考答案：

核心是所有任务查询、更新、删除都带上当前登录用户 ID。

JWT 中间件解析 token 后，把 user_id 放入 Gin Context。

handler 取出 user_id，传给 service。

repository 查询时：

```go
db.Where("id = ? AND user_id = ?", taskID, userID).First(&task)
```

更新时：

```go
db.Where("id = ? AND user_id = ?", taskID, userID).Updates(...)
```

不要只按 taskID 操作，否则用户可能访问别人任务。

---

### 63. 注册接口如何保证密码安全？

参考答案：

注册时不保存明文密码，而是使用 bcrypt 哈希：

```go
hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
```

数据库中保存 `password_hash`。

登录时：

```go
bcrypt.CompareHashAndPassword([]byte(user.PasswordHash), []byte(input.Password))
```

响应中永远不返回 password_hash。

日志中也不打印密码。

---

### 64. 登录接口如何设计？

参考答案：

流程：

```text
绑定请求参数。
检查登录失败限制。
根据 email 查询用户。
比较密码。
生成 JWT。
清除登录失败计数。
返回 token。
```

错误处理：

- 用户不存在和密码错误都返回“邮箱或密码错误”，避免泄露用户是否存在。
- 连续失败超过限制返回 429。
- 系统错误返回 500。

---

### 65. 任务状态如何设计？

参考答案：

可以定义枚举：

```text
todo
doing
done
cancelled
```

在请求 DTO 中校验：

```go
Status string `json:"status" binding:"omitempty,oneof=todo doing done cancelled"`
```

service 中也可以再做业务校验，避免绕过 HTTP 层时出现非法状态。

---

### 66. 项目中哪些地方用了 Redis？

参考答案：

可以回答：

> 我主要在两个场景使用 Redis。第一是用户 profile 或任务详情缓存，减少重复数据库查询；第二是登录失败次数限制，防止暴力破解。缓存 key 有统一前缀并设置 TTL。更新数据后会删除对应缓存，避免读到旧数据。

补充：

> 对于缓存场景，Redis 失败时可以降级查数据库；对于登录限制场景，要根据安全要求决定是否拒绝登录。

---

### 67. 项目如何部署？

参考答案：

项目使用 Docker Compose 本地部署，包含：

- app
- mysql
- redis
- nginx

app 镜像使用多阶段 Dockerfile 构建。

配置通过环境变量注入。

MySQL、Redis、uploads 使用 volume 持久化。

通过 `/healthz` 和 `/readyz` 做健康检查。

---

## 十二、综合系统设计题

### 68. 如果让你设计一个短链接系统，你会怎么设计？

参考答案：

核心功能：

- 创建短链接。
- 根据短码跳转。
- 统计访问次数。
- 查询短链接详情。

接口：

```text
POST /api/v1/links
GET /:code
GET /api/v1/links/:id
```

表设计：

```text
id
code
original_url
visit_count
created_at
updated_at
```

关键点：

- code 唯一索引。
- 短码生成要处理冲突。
- 跳转接口读多写少，适合 Redis 缓存。
- 访问次数可以异步统计。
- original_url 要校验合法性。

---

### 69. 如果接口突然变慢，你怎么排查？

参考答案：

排查顺序：

1. 看日志，确认是哪个接口慢。
2. 查看请求耗时、状态码、request_id。
3. 判断是否数据库慢。
4. 打开 SQL 日志或慢查询。
5. 检查 Redis 是否异常。
6. 检查外部 API 是否慢。
7. 查看系统 CPU、内存、goroutine。
8. 用 pprof 分析。

如果是数据库：

- 看 SQL。
- 看索引。
- 用 EXPLAIN。
- 检查分页是否深分页。

如果是 Redis：

- 看网络。
- 看 big key。
- 看命中率。

---

### 70. 如何设计一个安全的文件上传接口？

参考答案：

要限制：

- 文件大小。
- 文件扩展名。
- MIME 类型。
- 保存目录。
- 文件名。
- 访问权限。

不要直接使用用户上传的原始文件名。

推荐生成：

```text
userID_timestamp_random.ext
```

保存后返回 URL。

生产中建议使用对象存储。

Nginx 和应用层都要限制大小：

```nginx
client_max_body_size 5m;
```

Gin handler 中也要检查 `file.Size`。

---

## 十三、开放性与行为面试题

### 71. 你遇到过最难排查的问题是什么？

参考回答思路：

可以选择一个真实或练习中的问题，比如 Docker 容器连接 MySQL 失败。

回答结构：

```text
现象：app 容器启动失败，日志显示 connection refused。
排查：检查 MySQL 容器正常，检查 app 环境变量，发现 DSN 使用 127.0.0.1。
原因：容器内 localhost 指向 app 容器自身，不是 MySQL 容器。
解决：改成 mysql:3306。
复盘：README 中补充 Docker 网络说明，配置通过环境变量管理。
```

---

### 72. 你如何学习一个新技术？

参考答案：

可以这样答：

> 我通常先明确它解决什么问题，然后做一个最小可运行 demo，再把它接入已有项目，最后整理排错清单和最佳实践。比如学习 Redis 时，我不是只看命令，而是在 Gin 项目里实现 profile 缓存和登录失败限流，理解 key 设计、TTL、缓存一致性和降级策略。

---

### 73. 如果项目 deadline 很紧，你如何保证质量？

参考答案：

可以回答：

- 先明确核心路径。
- 优先完成最小可用功能。
- 保证关键错误处理。
- 保留测试请求集合。
- 对核心 service 写测试。
- 留下清晰 TODO 和风险说明。

强调：

> 我不会为了赶进度完全放弃边界和错误处理，尤其是认证、权限、数据库写入这类核心路径。

---

## 十四、面试前复习清单

面试前请确认你能讲清楚：

- [ ] Go 的 slice、map、goroutine、channel、context。
- [ ] Gin 路由、参数绑定、中间件。
- [ ] JWT 登录认证流程。
- [ ] 401 和 403 的区别。
- [ ] handler/service/repository 分层。
- [ ] DTO 和 Model 的区别。
- [ ] GORM CRUD、软删除、事务、唯一索引。
- [ ] Redis 缓存、TTL、登录失败限制。
- [ ] handler 测试和 service 测试。
- [ ] Dockerfile 和 Docker Compose。
- [ ] 容器中 localhost 的含义。
- [ ] 任务管理系统完整链路。
- [ ] 用户只能访问自己任务的实现。
- [ ] 常见线上排错思路。

---

## 十五、回答问题的模板

遇到技术问题，可以按这个模板答：

```text
1. 先给结论。
2. 解释原理。
3. 给代码或项目例子。
4. 补充边界和风险。
```

示例：

问题：为什么 service 不应该依赖 Gin？

回答：

```text
结论：service 是业务层，不应该依赖 Gin 这种 HTTP 框架。
原理：如果 service 接收 *gin.Context，业务逻辑会和 Web 框架耦合，测试也困难。
项目例子：我的任务系统中 handler 从 Gin Context 取 user_id，再把 context.Context、userID 和请求 DTO 传给 service。
边界：如果需要请求取消信号，传标准库 context.Context，而不是传 *gin.Context。
```

---

## 十六、最后建议

面试官通常不只想听“标准答案”，更想知道你是否真的做过、踩过坑、能排查。

所以复习时要把每个问题都和自己的项目联系起来：

```text
我在任务管理系统里怎么做的？
为什么这么做？
如果项目变大，会有什么问题？
下一步如何优化？
```

能回答这些，你的表达就会从“背知识点”变成“像一个真正写过后端项目的人”。
