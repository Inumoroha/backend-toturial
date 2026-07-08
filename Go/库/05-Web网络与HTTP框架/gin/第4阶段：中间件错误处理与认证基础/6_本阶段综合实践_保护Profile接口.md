# 6. 综合实践：保护 Profile 接口

本节目标：把本阶段内容串成一个完整认证闭环，实现注册、登录、JWT 认证中间件、Profile 私有接口和 Admin 权限接口。

这节不是单独知识点，而是综合练习。你要把中间件、统一响应、JWT、bcrypt、路由组和角色授权组合起来。

---

## 一、最终接口

本节目标接口：

```text
POST /api/v1/auth/register
POST /api/v1/auth/login
GET  /api/v1/profile
GET  /api/v1/admin/ping
```

接口权限：

```text
/auth/register 公开
/auth/login    公开
/profile       需要登录
/admin/ping    需要登录，并且 role=admin
```

---

## 二、用户模型

学习阶段可以先用内存 map：

```go
type User struct {
    ID           uint   `json:"id"`
    Username     string `json:"username"`
    Email        string `json:"email"`
    PasswordHash string `json:"-"`
    Role         string `json:"role"`
}

var users = map[uint]User{}
var nextUserID uint = 1
```

邮箱查找函数：

```go
func findUserByEmail(email string) (User, bool) {
    for _, user := range users {
        if user.Email == email {
            return user, true
        }
    }
    return User{}, false
}
```

真实项目中这些逻辑会放到 repository。

---

## 三、密码加密

安装：

```bash
go get golang.org/x/crypto/bcrypt
```

函数：

```go
func HashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    if err != nil {
        return "", err
    }
    return string(bytes), nil
}

func CheckPassword(password, hash string) bool {
    return bcrypt.CompareHashAndPassword([]byte(hash), []byte(password)) == nil
}
```

注册时保存 hash，不保存明文密码。

---

## 四、注册接口

请求结构体：

```go
type RegisterRequest struct {
    Username string `json:"username" binding:"required,min=2,max=32"`
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required,min=8,max=64"`
}
```

handler：

```go
func register(c *gin.Context) {
    var req RegisterRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.FailWithData(c, http.StatusBadRequest, response.CodeInvalidRequest, "invalid request", response.FormatValidationError(err))
        return
    }

    if _, exists := findUserByEmail(req.Email); exists {
        response.Fail(c, http.StatusConflict, response.CodeConflict, "email already exists")
        return
    }

    hash, err := HashPassword(req.Password)
    if err != nil {
        response.Fail(c, http.StatusInternalServerError, response.CodeInternalError, "internal server error")
        return
    }

    user := User{
        ID:           nextUserID,
        Username:     req.Username,
        Email:        req.Email,
        PasswordHash: hash,
        Role:         "user",
    }
    users[user.ID] = user
    nextUserID++

    response.Created(c, user)
}
```

---

## 五、登录接口

请求结构体：

```go
type LoginRequest struct {
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required"`
}
```

handler：

```go
func login(c *gin.Context) {
    var req LoginRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.FailWithData(c, http.StatusBadRequest, response.CodeInvalidRequest, "invalid request", response.FormatValidationError(err))
        return
    }

    user, exists := findUserByEmail(req.Email)
    if !exists || !CheckPassword(req.Password, user.PasswordHash) {
        response.Fail(c, http.StatusUnauthorized, response.CodeUnauthorized, "invalid email or password")
        return
    }

    token, err := GenerateToken(user.ID, user.Email, user.Role, jwtSecret, 24)
    if err != nil {
        response.Fail(c, http.StatusInternalServerError, response.CodeInternalError, "internal server error")
        return
    }

    response.OK(c, gin.H{"token": token})
}
```

登录失败统一返回：

```text
invalid email or password
```

不要区分邮箱不存在还是密码错误。

---

## 六、Profile 接口

```go
func profile(c *gin.Context) {
    userID := c.GetUint("user_id")

    user, exists := users[userID]
    if !exists {
        response.Fail(c, http.StatusNotFound, response.CodeNotFound, "user not found")
        return
    }

    response.OK(c, user)
}
```

这里的 `user_id` 来自认证中间件，不来自请求参数。

这是安全边界：

```text
用户不能通过传 user_id 查询别人信息。
```

---

## 七、Admin 接口

```go
func adminPing(c *gin.Context) {
    response.OK(c, gin.H{"message": "admin pong"})
}
```

路由中用 `RequireRole("admin")` 保护。

学习阶段可以手动把某个用户的 Role 改成 `admin` 测试。

---

## 八、路由注册

```go
const jwtSecret = "dev-secret"

func main() {
    r := gin.New()
    r.Use(middleware.RequestID())
    r.Use(middleware.AccessLog())
    r.Use(middleware.Recovery())

    api := r.Group("/api/v1")
    {
        api.POST("/auth/register", register)
        api.POST("/auth/login", login)

        private := api.Group("")
        private.Use(Auth(jwtSecret))
        {
            private.GET("/profile", profile)
        }

        admin := api.Group("/admin")
        admin.Use(Auth(jwtSecret), RequireRole("admin"))
        {
            admin.GET("/ping", adminPing)
        }
    }

    r.Run(":8080")
}
```

注意：

```text
register 和 login 不要放进 Auth 中间件。
profile 必须放进 Auth 中间件。
admin/ping 必须同时放进 Auth 和 RequireRole。
```

---

## 九、测试顺序

建议按顺序测试：

1. 不带 token 访问 `/profile`，应该 401。
2. 注册用户。
3. 用错误密码登录，应该 401。
4. 用正确密码登录，拿到 token。
5. 带 token 访问 `/profile`，应该 200。
6. 带错误 token 访问 `/profile`，应该 401。
7. 普通用户访问 `/admin/ping`，应该 403。

---

## 十、常见问题

### 1. 登录成功后 profile 还是 401

检查：

- 请求头是否写成 `Authorization: Bearer token`。
- token 是否复制完整。
- Auth 中间件使用的 secret 是否和生成 token 的 secret 一致。

### 2. profile 里 user_id 是 0

检查 Auth 中间件是否：

```go
c.Set("user_id", claims.UserID)
```

### 3. 密码明明正确但登录失败

检查注册时是否保存了 hash，登录时是否用 `CheckPassword`。

### 4. admin 接口一直 403

检查当前用户 role 是否真的是 `admin`。

---

## 十一、本节验收

完成后你应该能够做到：

- 注册用户，密码不明文保存。
- 登录成功返回 JWT。
- 不带 token 访问 profile 返回 401。
- 错误 token 返回 401。
- 正确 token 返回当前用户信息。
- 普通用户访问 admin 接口返回 403。
- 认证失败和权限失败都使用统一响应格式。

---

## 十二、复盘问题

请回答：

- register 和 login 为什么是公开接口？
- profile 为什么不接收 user_id 参数？
- JWT secret 不一致会造成什么问题？
- 认证失败为什么是 401？
- 授权失败为什么是 403？
- 哪些代码后续应该拆到 handler、service、repository？

