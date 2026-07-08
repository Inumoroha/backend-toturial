# 6. 登录接口与 JWT

本节目标：实现 `POST /api/v1/auth/login`，用户登录成功后返回 JWT。

---

## 一、登录流程

```text
读取 JSON
→ 校验 email/password
→ 根据 email 查询用户
→ bcrypt 校验密码
→ 生成 JWT
→ 返回 token
```

登录失败时不要告诉用户是“邮箱不存在”还是“密码错误”，统一返回：

```text
invalid email or password
```

这样可以减少账号枚举风险。

---

## 二、安装 JWT 库

```bash
go get github.com/golang-jwt/jwt/v5
```

---

## 三、JWT 工具函数

`internal/service/jwt.go`：

```go
package service

import (
    "errors"
    "time"

    "github.com/golang-jwt/jwt/v5"
)

type Claims struct {
    UserID uint   `json:"user_id"`
    Email  string `json:"email"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

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

---

## 四、AuthService 增加登录

```go
var ErrInvalidCredentials = errors.New("invalid credentials")

type AuthService struct {
    userRepo    *repository.UserRepository
    jwtSecret   string
    expireHours int
}

func NewAuthService(userRepo *repository.UserRepository, jwtSecret string, expireHours int) *AuthService {
    return &AuthService{
        userRepo:    userRepo,
        jwtSecret:   jwtSecret,
        expireHours: expireHours,
    }
}

type LoginInput struct {
    Email    string
    Password string
}

func (s *AuthService) Login(input LoginInput) (string, error) {
    user, err := s.userRepo.GetByEmail(input.Email)
    if errors.Is(err, repository.ErrUserNotFound) {
        return "", ErrInvalidCredentials
    }
    if err != nil {
        return "", err
    }

    if !CheckPassword(input.Password, user.PasswordHash) {
        return "", ErrInvalidCredentials
    }

    return GenerateToken(user.ID, user.Email, user.Role, s.jwtSecret, s.expireHours)
}
```

如果你前一节已经写了 `NewAuthService`，需要按这里更新构造函数。

---

## 五、AuthHandler 增加 Login

```go
type LoginRequest struct {
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required"`
}

func (h *AuthHandler) Login(c *gin.Context) {
    var req LoginRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.Fail(c, http.StatusBadRequest, response.CodeInvalidRequest, "invalid request")
        return
    }

    token, err := h.service.Login(service.LoginInput{
        Email:    req.Email,
        Password: req.Password,
    })
    if errors.Is(err, service.ErrInvalidCredentials) {
        response.Fail(c, http.StatusUnauthorized, response.CodeUnauthorized, "invalid email or password")
        return
    }
    if err != nil {
        response.Fail(c, http.StatusInternalServerError, response.CodeInternalError, "internal server error")
        return
    }

    response.OK(c, gin.H{
        "token": token,
    })
}
```

---

## 六、路由注册

```go
api.POST("/auth/register", authHandler.Register)
api.POST("/auth/login", authHandler.Login)
```

---

## 七、测试请求

```http
### login
POST http://localhost:8080/api/v1/auth/login
Content-Type: application/json

{
  "email": "alice@example.com",
  "password": "12345678"
}
```

响应：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "token": "..."
  }
}
```

---

## 八、常见问题

### 1. 登录一直失败

检查：

- 注册时是否真的保存了 bcrypt hash。
- 登录时是否使用 `CheckPassword`。
- 邮箱是否大小写或空格不同。

### 2. token 生成失败

检查：

- JWT secret 是否为空。
- `expireHours` 是否正确读取。

### 3. token 很长是否正常

正常。JWT 本身就是一段较长字符串。

---

## 九、本节验收

你应该能够回答：

- 登录为什么不区分邮箱不存在和密码错误？
- JWT claims 中放了哪些信息？
- token 为什么要设置过期时间？
- JWT secret 为什么不能泄露？
- 登录成功后前端应该如何携带 token？

