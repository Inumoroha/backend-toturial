# 4. Handler 测试实践

本节目标：使用 Go 标准库 `net/http/httptest` 测试 Gin handler 的 HTTP 行为。

handler 测试是 Gin 项目里最应该优先掌握的测试之一。因为 handler 是外部请求进入系统的入口，它的行为直接影响前端、客户端和其他调用方。

---

## 一、handler 测试关注什么

handler 测试应该关注 HTTP 层面的行为：

- 请求方法是否正确。
- 请求路径是否正确。
- JSON 请求体能否正确绑定。
- 参数错误是否返回 `400`。
- 未认证是否返回 `401`。
- 权限不足是否返回 `403`。
- 资源不存在是否返回 `404`。
- 成功时是否返回预期 JSON。

handler 测试不应该把重点放在数据库细节上。数据库读写应该由 repository 或集成测试覆盖。

---

## 二、最小测试示例

先看一个最小例子：

```go
func TestPing(t *testing.T) {
    gin.SetMode(gin.TestMode)

    r := gin.New()
    r.GET("/ping", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"message": "pong"})
    })

    req := httptest.NewRequest(http.MethodGet, "/ping", nil)
    w := httptest.NewRecorder()

    r.ServeHTTP(w, req)

    if w.Code != http.StatusOK {
        t.Fatalf("want status %d, got %d", http.StatusOK, w.Code)
    }

    var body map[string]string
    if err := json.Unmarshal(w.Body.Bytes(), &body); err != nil {
        t.Fatalf("unmarshal response: %v", err)
    }

    if body["message"] != "pong" {
        t.Fatalf("want pong, got %q", body["message"])
    }
}
```

需要导入：

```go
import (
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/gin-gonic/gin"
)
```

关键对象解释：

- `httptest.NewRequest`：构造一个 HTTP 请求。
- `httptest.NewRecorder`：记录 handler 写出的响应。
- `r.ServeHTTP(w, req)`：让 Gin 处理这次请求。
- `w.Code`：响应状态码。
- `w.Body`：响应体。

---

## 三、准备一个可测试的 handler

假设用户注册 handler 长这样：

```go
type UserService interface {
    Register(ctx context.Context, req service.RegisterRequest) (*service.UserDTO, error)
}

type UserHandler struct {
    userService UserService
}

func NewUserHandler(userService UserService) *UserHandler {
    return &UserHandler{userService: userService}
}

func (h *UserHandler) Register(c *gin.Context) {
    var req RegisterRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.Fail(c, http.StatusBadRequest, "INVALID_PARAMS", "参数错误")
        return
    }

    user, err := h.userService.Register(c.Request.Context(), service.RegisterRequest{
        Email:    req.Email,
        Password: req.Password,
        Nickname: req.Nickname,
    })
    if err != nil {
        response.Fail(c, http.StatusBadRequest, "REGISTER_FAILED", err.Error())
        return
    }

    response.OK(c, user)
}
```

handler 依赖的是接口 `UserService`，而不是具体 service 实现。这样测试时就可以传入 fake service。

---

## 四、编写 fake service

在 `user_handler_test.go` 中写：

```go
type fakeUserService struct {
    registerFunc func(ctx context.Context, req service.RegisterRequest) (*service.UserDTO, error)
}

func (f *fakeUserService) Register(ctx context.Context, req service.RegisterRequest) (*service.UserDTO, error) {
    if f.registerFunc != nil {
        return f.registerFunc(ctx, req)
    }
    return &service.UserDTO{
        ID:    1,
        Email: req.Email,
    }, nil
}
```

fake 的作用是让 handler 测试不依赖真实数据库、不依赖真实 service 复杂逻辑。

---

## 五、测试注册成功

```go
func TestUserHandler_Register_Success(t *testing.T) {
    gin.SetMode(gin.TestMode)

    svc := &fakeUserService{
        registerFunc: func(ctx context.Context, req service.RegisterRequest) (*service.UserDTO, error) {
            if req.Email != "a@example.com" {
                t.Fatalf("unexpected email: %s", req.Email)
            }

            return &service.UserDTO{
                ID:    1,
                Email: req.Email,
            }, nil
        },
    }

    h := NewUserHandler(svc)
    r := gin.New()
    r.POST("/api/v1/register", h.Register)

    body := strings.NewReader(`{
        "email": "a@example.com",
        "password": "123456",
        "nickname": "alice"
    }`)
    req := httptest.NewRequest(http.MethodPost, "/api/v1/register", body)
    req.Header.Set("Content-Type", "application/json")
    w := httptest.NewRecorder()

    r.ServeHTTP(w, req)

    if w.Code != http.StatusOK {
        t.Fatalf("want status 200, got %d, body=%s", w.Code, w.Body.String())
    }
}
```

这个测试验证了：

- 路由能匹配。
- JSON 能绑定成功。
- handler 调用了 service。
- 成功时返回 `200`。

---

## 六、测试参数错误

```go
func TestUserHandler_Register_InvalidJSON(t *testing.T) {
    gin.SetMode(gin.TestMode)

    h := NewUserHandler(&fakeUserService{})
    r := gin.New()
    r.POST("/api/v1/register", h.Register)

    body := strings.NewReader(`{"email": ""}`)
    req := httptest.NewRequest(http.MethodPost, "/api/v1/register", body)
    req.Header.Set("Content-Type", "application/json")
    w := httptest.NewRecorder()

    r.ServeHTTP(w, req)

    if w.Code != http.StatusBadRequest {
        t.Fatalf("want status 400, got %d", w.Code)
    }
}
```

要让这个测试有效，请确保请求结构体有校验规则：

```go
type RegisterRequest struct {
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required,min=6"`
    Nickname string `json:"nickname" binding:"omitempty,max=20"`
}
```

---

## 七、测试未登录接口

假设 `/api/v1/profile` 需要 JWT 中间件：

```go
func TestProfile_Unauthorized(t *testing.T) {
    gin.SetMode(gin.TestMode)

    r := gin.New()
    r.GET("/api/v1/profile", middleware.Auth("test-secret"), func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"message": "ok"})
    })

    req := httptest.NewRequest(http.MethodGet, "/api/v1/profile", nil)
    w := httptest.NewRecorder()

    r.ServeHTTP(w, req)

    if w.Code != http.StatusUnauthorized {
        t.Fatalf("want status 401, got %d", w.Code)
    }
}
```

认证类测试的重点不是 JWT 算法本身，而是接口在没有 token 时是否被拦截。

---

## 八、解析统一响应

如果你的统一响应结构是：

```go
type Response struct {
    Code    string      `json:"code"`
    Message string      `json:"message"`
    Data    interface{} `json:"data,omitempty"`
}
```

测试中可以定义：

```go
type testResponse struct {
    Code    string          `json:"code"`
    Message string          `json:"message"`
    Data    json.RawMessage `json:"data"`
}
```

然后解析：

```go
var resp testResponse
if err := json.Unmarshal(w.Body.Bytes(), &resp); err != nil {
    t.Fatalf("unmarshal response: %v", err)
}

if resp.Code != "OK" {
    t.Fatalf("want code OK, got %s", resp.Code)
}
```

不要只检查状态码。接口测试最好同时检查状态码和响应结构。

---

## 九、常见问题

### 1. 测试输出很多 Gin debug 日志

在测试开头设置：

```go
gin.SetMode(gin.TestMode)
```

### 2. `ShouldBindJSON` 一直失败

检查是否设置请求头：

```go
req.Header.Set("Content-Type", "application/json")
```

也要检查 JSON 字段名和结构体 tag 是否一致。

### 3. fake service 没有被调用

检查路由路径和请求方法是否一致。例如注册的是 `POST`，测试却用了 `GET`。

---

## 十、练习

请为你的项目补充这些 handler 测试：

1. `GET /ping` 返回 `200`。
2. `POST /api/v1/register` 参数正确时返回成功。
3. `POST /api/v1/register` 邮箱格式错误时返回 `400`。
4. `POST /api/v1/login` 密码为空时返回 `400`。
5. `GET /api/v1/profile` 不带 token 时返回 `401`。

---

## 十一、验收标准

本节完成后，请运行：

```bash
go test ./internal/handler -v
```

确认：

- 测试全部通过。
- 每个核心 handler 至少有成功和失败用例。
- 测试中没有连接真实数据库。
- 统一响应结构被检查。

handler 测试过关后，再进入 service 测试。
