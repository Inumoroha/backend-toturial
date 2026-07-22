# 3. JWT 认证与角色授权

本节目标：理解 JWT 登录认证流程，掌握 token 的生成、解析、认证中间件和基础角色授权。完成本节后，你应该能保护需要登录的接口，并能区分认证和授权。

---

## 一、认证和授权的区别

认证回答的问题是：

```text
你是谁？
```

例如用户登录后，服务端通过 token 知道：

```text
当前请求来自 user_id = 1 的用户。
```

授权回答的问题是：

```text
你能做什么？
```

例如：

```text
普通用户不能访问管理员接口。
用户只能修改自己的任务。
```

简单区分：

```text
认证失败 -> 401
授权失败 -> 403
```

---

## 二、JWT 是什么

JWT 是 JSON Web Token。

它通常长这样：

```text
xxxxx.yyyyy.zzzzz
```

由三部分组成：

```text
Header.Payload.Signature
```

你不需要手动拼接 JWT，使用库即可。

JWT 常用于无状态认证：

```text
用户登录成功 -> 服务端签发 token -> 客户端保存 token -> 后续请求携带 token -> 服务端解析 token
```

---

## 三、安装 JWT 库

```bash
go get github.com/golang-jwt/jwt/v5
```

---

## 四、定义 Claims

Claims 是你放进 token 的用户信息。

```go
type Claims struct {
    UserID uint   `json:"user_id"`
    Email  string `json:"email"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}
```

字段说明：

```text
UserID
当前用户 ID。

Email
当前用户邮箱。

Role
当前用户角色，例如 user/admin。

RegisteredClaims
JWT 标准字段，例如过期时间、签发时间。
```

不要把密码、身份证、手机号等敏感信息放进 JWT。

---

## 五、生成 token

```go
func GenerateToken(userID uint, email, role, secret string, expireHours int) (string, error) {
    claims := Claims{
        UserID: userID,
        Email:  email,
        Role:   role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(time.Duration(expireHours) * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(secret))
}
```

参数说明：

- `userID`：用户 ID。
- `email`：用户邮箱。
- `role`：用户角色。
- `secret`：签名密钥。
- `expireHours`：过期小时数。

`secret` 不能泄露，也不要硬编码生产密钥。

---

## 六、解析 token

```go
func ParseToken(tokenString, secret string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        return []byte(secret), nil
    })
    if err != nil {
        return nil, err
    }

    claims, ok := token.Claims.(*Claims)
    if !ok || !token.Valid {
        return nil, errors.New("invalid token")
    }

    return claims, nil
}
```

需要导入：

```go
import (
    "errors"
    "time"

    "github.com/golang-jwt/jwt/v5"
)
```

---

## 七、客户端如何携带 token

推荐使用 Header：

```http
Authorization: Bearer token
```

不要把 token 放在 query 参数里：

```text
/api/v1/profile?token=xxx
```

原因：

- query 参数容易进入日志。
- query 参数容易被浏览器历史记录保存。
- Header 是更标准的认证信息携带方式。

---

## 八、认证中间件

```go
func Auth(secret string) gin.HandlerFunc {
    return func(c *gin.Context) {
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

        claims, err := ParseToken(parts[1], secret)
        if err != nil {
            response.Fail(c, http.StatusUnauthorized, response.CodeUnauthorized, "invalid token")
            c.Abort()
            return
        }

        c.Set("user_id", claims.UserID)
        c.Set("email", claims.Email)
        c.Set("role", claims.Role)

        c.Next()
    }
}
```

需要导入：

```go
import (
    "net/http"
    "strings"

    "github.com/gin-gonic/gin"
)
```

---

## 九、为什么要写入 gin.Context

中间件解析 token 后，把用户信息写入：

```go
c.Set("user_id", claims.UserID)
```

后续 handler 就可以读取：

```go
userID := c.GetUint("user_id")
```

这样 handler 不需要重复解析 token。

这也是中间件的价值：一次解析，后续共享。

---

## 十、路由组保护私有接口

公开接口：

```go
api.POST("/auth/register", authHandler.Register)
api.POST("/auth/login", authHandler.Login)
```

私有接口：

```go
private := api.Group("")
private.Use(Auth(secret))
{
    private.GET("/profile", userHandler.Profile)
    private.GET("/tasks", taskHandler.List)
}
```

不要把登录接口放进认证路由组，否则用户还没登录就无法登录。

---

## 十一、角色授权中间件

基础角色中间件：

```go
func RequireRole(role string) gin.HandlerFunc {
    return func(c *gin.Context) {
        currentRole, exists := c.Get("role")
        if !exists || currentRole != role {
            response.Fail(c, http.StatusForbidden, response.CodeForbidden, "forbidden")
            c.Abort()
            return
        }

        c.Next()
    }
}
```

管理员路由：

```go
admin := api.Group("/admin")
admin.Use(Auth(secret), RequireRole("admin"))
{
    admin.GET("/ping", func(c *gin.Context) {
        response.OK(c, gin.H{"message": "admin pong"})
    })
}
```

---

## 十二、401 和 403 的选择

没有 token：

```text
401 Unauthorized
```

token 无效：

```text
401 Unauthorized
```

token 正确，但角色不够：

```text
403 Forbidden
```

用户访问不属于自己的资源：

```text
403 或 404
```

两种都可以，取决于项目是否想隐藏资源存在性。关键是项目内保持一致。

---

## 十三、常见安全问题

### 1. JWT secret 写死在代码里

学习阶段可以临时写死，真实项目应该来自配置或环境变量。

### 2. token 不设置过期时间

不推荐。token 应该有过期时间。

### 3. 把密码放进 JWT

绝对不要。

### 4. 只靠前端隐藏按钮做权限

不安全。后端必须检查权限。

### 5. 登录失败提示太细

不推荐：

```text
email not found
```

更推荐：

```text
invalid email or password
```

避免暴露账号是否存在。

---

## 十四、本节练习

完成：

1. 安装 `github.com/golang-jwt/jwt/v5`。
2. 定义 `Claims`。
3. 实现 `GenerateToken`。
4. 实现 `ParseToken`。
5. 实现 `Auth(secret)` 中间件。
6. 实现 `RequireRole("admin")` 中间件。
7. 创建 `/api/v1/profile` 私有接口。
8. 创建 `/api/v1/admin/ping` 管理员接口。

---

## 十五、本节验收

你应该能够回答：

- 认证和授权有什么区别？
- JWT 通常由哪三部分组成？
- token 为什么要放在 Authorization Header？
- 为什么 JWT 里不要放敏感信息？
- 认证中间件为什么要把 user_id 写入 `gin.Context`？
- 401 和 403 分别适合什么场景？


