# 7. 认证中间件与 Profile 接口

本节目标：实现 JWT 认证中间件，并完成 `GET /api/v1/profile`。

---

## 一、认证中间件流程

```text
读取 Authorization header
→ 检查 Bearer 格式
→ 解析 JWT
→ 把 user_id/email/role 写入 gin.Context
→ 放行后续 handler
```

如果失败：

```text
返回 401
调用 c.Abort()
```

---

## 二、Authorization 格式

客户端请求私有接口时要带：

```http
Authorization: Bearer token
```

注意 `Bearer` 后面有一个空格。

---

## 三、认证中间件代码

`internal/middleware/auth.go`：

```go
package middleware

import (
    "net/http"
    "strings"

    "task-api/internal/response"
    "task-api/internal/service"

    "github.com/gin-gonic/gin"
)

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

        claims, err := service.ParseToken(parts[1], secret)
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

---

## 四、UserRepository 增加 GetByID

```go
func (r *UserRepository) GetByID(id uint) (*model.User, error) {
    var user model.User
    err := r.db.First(&user, id).Error
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return nil, ErrUserNotFound
    }
    if err != nil {
        return nil, err
    }
    return &user, nil
}
```

---

## 五、UserService

`internal/service/user_service.go`：

```go
package service

import (
    "task-api/internal/model"
    "task-api/internal/repository"
)

type UserService struct {
    repo *repository.UserRepository
}

func NewUserService(repo *repository.UserRepository) *UserService {
    return &UserService{repo: repo}
}

func (s *UserService) GetProfile(userID uint) (*model.User, error) {
    return s.repo.GetByID(userID)
}
```

---

## 六、UserHandler

`internal/handler/user_handler.go`：

```go
package handler

import (
    "errors"
    "net/http"

    "task-api/internal/repository"
    "task-api/internal/response"
    "task-api/internal/service"

    "github.com/gin-gonic/gin"
)

type UserHandler struct {
    service *service.UserService
}

func NewUserHandler(service *service.UserService) *UserHandler {
    return &UserHandler{service: service}
}

func (h *UserHandler) Profile(c *gin.Context) {
    userID := c.GetUint("user_id")

    user, err := h.service.GetProfile(userID)
    if errors.Is(err, repository.ErrUserNotFound) {
        response.Fail(c, http.StatusNotFound, response.CodeNotFound, "user not found")
        return
    }
    if err != nil {
        response.Fail(c, http.StatusInternalServerError, response.CodeInternalError, "internal server error")
        return
    }

    response.OK(c, user)
}
```

---

## 七、路由分组

```go
api := r.Group("/api/v1")

api.POST("/auth/register", authHandler.Register)
api.POST("/auth/login", authHandler.Login)

private := api.Group("")
private.Use(middleware.Auth(cfg.JWT.Secret))
{
    private.GET("/profile", userHandler.Profile)
}
```

---

## 八、测试请求

不带 token：

```http
GET http://localhost:8080/api/v1/profile
```

应该返回 401。

带 token：

```http
GET http://localhost:8080/api/v1/profile
Authorization: Bearer your-token
```

应该返回用户信息。

---

## 九、常见问题

### 1. `c.GetUint("user_id")` 得到 0

检查中间件里是否写了：

```go
c.Set("user_id", claims.UserID)
```

以及路由是否真的挂载了中间件。

### 2. 认证失败后 handler 还执行

检查失败分支是否调用：

```go
c.Abort()
return
```

### 3. token 格式错误

确认请求头是：

```text
Authorization: Bearer token
```

不是只传 token。

---

## 十、本节验收

你应该能够回答：

- 认证中间件为什么要写入 `gin.Context`？
- `c.Abort()` 的作用是什么？
- 为什么私有接口要放到路由组里？
- 401 和 403 有什么区别？
- `/profile` 为什么不从请求参数里读取 user_id？

