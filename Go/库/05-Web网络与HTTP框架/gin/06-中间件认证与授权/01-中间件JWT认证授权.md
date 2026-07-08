# 06-01 中间件、JWT 认证与授权

本阶段目标：理解 Gin 中间件的执行机制，并实现注册、登录、JWT 认证和基础权限控制。

## 一、什么是中间件

中间件是请求进入 handler 前后执行的一段逻辑。常见用途：

- 记录请求日志。
- 处理跨域。
- 认证用户身份。
- 权限检查。
- 统计请求耗时。
- 捕获 panic。

Gin 中间件本质上也是一个函数：

```go
func(c *gin.Context)
```

## 二、最小中间件

```go
func RequestIDMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("X-Request-ID", "demo-request-id")
        c.Next()
    }
}
```

注册：

```go
r.Use(RequestIDMiddleware())
```

`c.Next()` 表示继续执行后续中间件和 handler。

如果你想终止请求：

```go
c.AbortWithStatusJSON(401, gin.H{
    "code": 40101,
    "message": "unauthorized",
})
return
```

## 三、注册与登录接口设计

接口：

```text
POST /api/v1/auth/register
POST /api/v1/auth/login
GET  /api/v1/profile
```

注册请求：

```json
{
  "username": "alice",
  "email": "alice@example.com",
  "password": "12345678"
}
```

登录请求：

```json
{
  "email": "alice@example.com",
  "password": "12345678"
}
```

登录响应：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "token": "jwt-token"
  }
}
```

## 四、密码加密

不要明文保存密码。

安装：

```bash
go get golang.org/x/crypto/bcrypt
```

加密：

```go
func HashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(bytes), err
}
```

校验：

```go
func CheckPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}
```

用户表只保存 `password_hash`，永远不要返回给前端。

## 五、生成 JWT

安装：

```bash
go get github.com/golang-jwt/jwt/v5
```

定义 claims：

```go
type Claims struct {
    UserID uint   `json:"user_id"`
    Email  string `json:"email"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}
```

生成 token：

```go
func GenerateToken(userID uint, email, role string, secret string) (string, error) {
    claims := Claims{
        UserID: userID,
        Email:  email,
        Role:   role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(secret))
}
```

解析 token：

```go
func ParseToken(tokenString string, secret string) (*Claims, error) {
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

## 六、认证中间件

客户端通常这样携带 token：

```text
Authorization: Bearer jwt-token
```

中间件：

```go
func AuthMiddleware(secret string) gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" {
            response.Fail(c, http.StatusUnauthorized, 40101, "missing token")
            c.Abort()
            return
        }

        parts := strings.SplitN(authHeader, " ", 2)
        if len(parts) != 2 || parts[0] != "Bearer" {
            response.Fail(c, http.StatusUnauthorized, 40101, "invalid token format")
            c.Abort()
            return
        }

        claims, err := ParseToken(parts[1], secret)
        if err != nil {
            response.Fail(c, http.StatusUnauthorized, 40101, "invalid token")
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

handler 中获取当前用户：

```go
userIDValue, exists := c.Get("user_id")
if !exists {
    response.Fail(c, http.StatusUnauthorized, 40101, "unauthorized")
    return
}

userID := userIDValue.(uint)
```

## 七、路由分组保护

```go
api := r.Group("/api/v1")

api.POST("/auth/register", authHandler.Register)
api.POST("/auth/login", authHandler.Login)

private := api.Group("")
private.Use(middleware.AuthMiddleware(secret))
{
    private.GET("/profile", userHandler.Profile)
    private.PUT("/profile", userHandler.UpdateProfile)
}
```

公开接口和需要登录的接口要分清楚。

## 八、角色权限控制

简单角色中间件：

```go
func RequireRole(role string) gin.HandlerFunc {
    return func(c *gin.Context) {
        currentRole, _ := c.Get("role")
        if currentRole != role {
            response.Fail(c, http.StatusForbidden, 40301, "forbidden")
            c.Abort()
            return
        }
        c.Next()
    }
}
```

使用：

```go
admin := api.Group("/admin")
admin.Use(AuthMiddleware(secret), RequireRole("admin"))
{
    admin.GET("/users", userHandler.AdminList)
}
```

## 九、常见安全问题

- 密码不能明文保存。
- JWT secret 不能写死在代码里。
- token 要设置过期时间。
- 登录失败不要提示“邮箱存在但密码错误”这类过细信息。
- 重要操作不能只依赖前端隐藏按钮，后端也必须鉴权。
- 生产环境必须使用 HTTPS。

## 十、阶段练习

完成：

- 注册接口。
- 登录接口。
- JWT 生成和解析。
- 认证中间件。
- 获取个人信息接口。
- 更新个人信息接口。
- 管理员角色接口。

要求：

- 密码使用 bcrypt。
- token 过期时间为 24 小时。
- 未登录访问 `/profile` 返回 `40101`。
- 普通用户访问管理员接口返回 `40301`。

## 十一、验收清单

你应该能够回答：

- `c.Next()` 和 `c.Abort()` 的区别是什么？
- JWT 由哪几部分组成？
- 为什么密码不能明文保存？
- 为什么认证信息要放到 `gin.Context`？
- 认证和授权有什么区别？

