# 8. 中间件、JWT 排错与验收清单

本节目标：把第4阶段最容易混淆的中间件执行顺序、JWT 认证、角色授权、CORS、Recovery、RequestID 和统一错误响应集中整理成一份排错清单。

第4阶段学完后，你不应该只会“写一个 JWT 中间件”。你还要能判断：

- 中间件为什么没有执行。
- `c.Next()` 前后分别适合做什么。
- 为什么没有 token 应该返回 `401`。
- 为什么权限不足应该返回 `403`。
- CORS 预检请求为什么失败。
- panic 后为什么响应格式不统一。
- 日志里为什么没有 request_id。

这些问题在真实项目里非常常见。

---

## 一、中间件注册顺序排错

Gin 中间件按注册顺序形成洋葱模型。

推荐顺序：

```go
r := gin.New()

r.Use(middleware.RequestID())
r.Use(middleware.AccessLog(logger))
r.Use(middleware.Recovery(logger))
r.Use(middleware.CORS())
```

认证中间件通常注册在需要保护的路由组上：

```go
api := r.Group("/api/v1")

api.POST("/login", userHandler.Login)

auth := api.Group("")
auth.Use(middleware.Auth(jwtManager))
{
    auth.GET("/profile", userHandler.Profile)
    auth.PUT("/profile", userHandler.UpdateProfile)
}
```

自查问题：

- 全局中间件是否注册在路由之前。
- 认证中间件是否只保护需要登录的接口。
- 登录、注册接口是否没有被认证中间件误伤。
- RequestID 是否在 AccessLog 前面。
- Recovery 是否覆盖所有业务路由。

---

## 二、`c.Next()` 与 `c.Abort()` 排错

中间件中最容易写错的是流程控制。

正常放行：

```go
func Demo() gin.HandlerFunc {
    return func(c *gin.Context) {
        // 请求进入 handler 前
        c.Next()
        // handler 执行后
    }
}
```

认证失败时应该中断：

```go
func Auth() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            response.Fail(c, http.StatusUnauthorized, response.CodeUnauthorized, "unauthorized")
            c.Abort()
            return
        }

        c.Next()
    }
}
```

常见错误：

```go
if token == "" {
    response.Fail(...)
}
c.Next()
```

这样即使认证失败，后面的 handler 仍然可能继续执行。

自查问题：

- 拒绝请求后是否调用 `c.Abort()`。
- `c.Abort()` 后是否立刻 `return`。
- 正常请求是否调用 `c.Next()`。
- 是否在 `c.Next()` 后读取最终状态码和耗时。

---

## 三、JWT Header 格式排错

推荐格式：

```http
Authorization: Bearer eyJhbGciOi...
```

中间件应检查：

```go
authHeader := c.GetHeader("Authorization")
if authHeader == "" {
    response.Fail(c, http.StatusUnauthorized, response.CodeUnauthorized, "missing token")
    c.Abort()
    return
}

parts := strings.SplitN(authHeader, " ", 2)
if len(parts) != 2 || parts[0] != "Bearer" {
    response.Fail(c, http.StatusUnauthorized, response.CodeUnauthorized, "invalid token format")
    c.Abort()
    return
}

tokenString := parts[1]
```

常见错误：

- 前端只传 token，没有 `Bearer ` 前缀。
- `Bearer` 后面没有空格。
- 后端直接把整个 Header 当 token 解析。
- token 过期和 token 格式错误没有区分日志。

对外响应可以都返回：

```json
{
  "code": 40101,
  "message": "unauthorized"
}
```

详细原因写日志即可。

---

## 四、401 与 403 排错

这两个状态码很容易混。

```text
401 Unauthorized：没有登录，或者 token 无效。
403 Forbidden：已经登录，但没有权限。
```

示例：

```text
不带 token 访问 /profile -> 401
普通用户访问 /admin/users -> 403
```

handler 或中间件中：

```go
response.Fail(c, http.StatusUnauthorized, response.CodeUnauthorized, "unauthorized")
```

角色不足：

```go
response.Fail(c, http.StatusForbidden, response.CodeForbidden, "forbidden")
```

自查问题：

- 没登录是否返回 `401`。
- token 过期是否返回 `401`。
- 角色不足是否返回 `403`。
- 是否没有把权限不足错误写成 `500`。

---

## 五、用户信息写入 Context 排错

JWT 解析成功后，应该把当前用户信息写入 Gin Context：

```go
c.Set("user_id", claims.UserID)
c.Set("role", claims.Role)
```

读取时要做类型判断：

```go
func CurrentUserID(c *gin.Context) (uint, bool) {
    value, exists := c.Get("user_id")
    if !exists {
        return 0, false
    }

    userID, ok := value.(uint)
    return userID, ok
}
```

不要直接强转：

```go
userID := c.MustGet("user_id").(uint)
```

学习阶段能跑，但一旦中间件漏注册，就可能 panic。

自查问题：

- JWT 解析成功后是否写入 `user_id`。
- handler 是否从 Context 读取当前用户。
- 类型断言失败时是否有明确错误响应。
- 是否避免在 handler 中重复解析 token。

---

## 六、CORS 排错

浏览器跨域请求失败时，后端日志可能看不到真正业务请求，因为浏览器先发的是预检请求：

```http
OPTIONS /api/v1/profile
Origin: http://localhost:5173
Access-Control-Request-Method: GET
```

CORS 中间件要处理 OPTIONS：

```go
func CORS() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("Access-Control-Allow-Origin", "http://localhost:5173")
        c.Header("Access-Control-Allow-Methods", "GET,POST,PUT,PATCH,DELETE,OPTIONS")
        c.Header("Access-Control-Allow-Headers", "Content-Type,Authorization")

        if c.Request.Method == http.MethodOptions {
            c.AbortWithStatus(http.StatusNoContent)
            return
        }

        c.Next()
    }
}
```

自查问题：

- 是否允许前端实际 origin。
- 是否允许 `Authorization` Header。
- 是否处理 OPTIONS。
- CORS 中间件是否注册在认证中间件之前。
- 是否在生产环境避免无脑 `*`。

---

## 七、Recovery 排错

Gin 默认 Recovery 会返回文本或默认响应，不一定符合你的统一响应。建议自定义：

```go
func Recovery(log *zap.Logger) gin.HandlerFunc {
    return gin.CustomRecovery(func(c *gin.Context, recovered interface{}) {
        log.Error("panic recovered",
            zap.Any("panic", recovered),
            zap.String("path", c.Request.URL.Path),
        )

        response.Fail(c, http.StatusInternalServerError, response.CodeInternalError, "internal server error")
    })
}
```

自查问题：

- panic 是否被捕获。
- panic 是否输出日志。
- 对外是否返回统一 JSON。
- 是否没有把 panic 细节直接返回给用户。

---

## 八、RequestID 与请求日志排错

RequestID 中间件应该：

```go
requestID := c.GetHeader("X-Request-ID")
if requestID == "" {
    requestID = uuid.NewString()
}

c.Set("request_id", requestID)
c.Header("X-Request-ID", requestID)
```

请求日志应该在 `c.Next()` 后记录：

```go
start := time.Now()
c.Next()

logger.Info("request completed",
    zap.String("request_id", requestID),
    zap.String("method", c.Request.Method),
    zap.String("path", c.Request.URL.Path),
    zap.Int("status", c.Writer.Status()),
    zap.Duration("latency", time.Since(start)),
)
```

自查问题：

- 响应头是否有 `X-Request-ID`。
- 日志是否包含 request_id。
- 4xx、5xx 请求是否也有日志。
- AccessLog 是否注册在业务路由之前。

---

## 九、认证接口测试清单

`requests.http` 中至少保留：

```http
### profile without token
GET http://localhost:8080/api/v1/profile

### profile invalid token format
GET http://localhost:8080/api/v1/profile
Authorization: wrong-token

### profile invalid bearer token
GET http://localhost:8080/api/v1/profile
Authorization: Bearer wrong-token

### profile with token
GET http://localhost:8080/api/v1/profile
Authorization: Bearer {{token}}

### admin api as normal user
GET http://localhost:8080/api/v1/admin/users
Authorization: Bearer {{user_token}}
```

期望：

```text
无 token -> 401
格式错误 -> 401
token 无效 -> 401
普通用户访问管理员接口 -> 403
合法 token 访问 profile -> 200
```

---

## 十、最终验收清单

请逐项确认：

- [ ] 全局中间件注册顺序清楚。
- [ ] 登录、注册接口没有被 JWT 中间件保护。
- [ ] 需要登录的接口被 JWT 中间件保护。
- [ ] 认证失败后调用 `c.Abort()` 并 `return`。
- [ ] JWT Header 使用 `Bearer token` 格式。
- [ ] 无 token、格式错误、过期 token 返回 `401`。
- [ ] 角色不足返回 `403`。
- [ ] 当前用户 ID 写入 Gin Context。
- [ ] CORS 支持 OPTIONS 预检。
- [ ] CORS 允许 `Authorization` Header。
- [ ] panic 被 Recovery 捕获。
- [ ] panic 响应仍是统一 JSON。
- [ ] 每个请求都有 `X-Request-ID`。
- [ ] 请求日志包含 method、path、status、latency、request_id。
- [ ] `requests.http` 覆盖认证成功和失败场景。

这些全部完成后，第4阶段才算真正掌握。下一阶段会把这些接口和中间件放进更清晰的目录结构中，开始做分层重构和工程化组织。

