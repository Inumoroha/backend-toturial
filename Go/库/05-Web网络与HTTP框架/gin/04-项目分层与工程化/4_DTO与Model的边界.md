# 4. DTO 与 Model 的边界

本节目标：理解 HTTP 请求结构体、service input、业务 model、数据库 model 之间的区别。很多 Gin 初学者会把一个结构体到处复用，短期省事，长期会让代码越来越混乱。

---

## 一、先看一个常见错误

假设数据库模型是：

```go
type User struct {
    ID           uint   `json:"id"`
    Username     string `json:"username"`
    Email        string `json:"email"`
    PasswordHash string `json:"password_hash"`
    Role         string `json:"role"`
}
```

如果你直接用它接收注册请求：

```go
var user User
c.ShouldBindJSON(&user)
```

就会出现几个问题：

- 前端可能传 `id`。
- 前端可能传 `role`。
- 前端可能传 `password_hash`。
- 响应时可能把 `password_hash` 返回给前端。

这非常危险。

---

## 二、什么是 DTO

DTO 是 Data Transfer Object，简单理解就是“传输数据用的结构体”。

在 Gin 项目里，常见 DTO 包括：

```text
CreateUserRequest
UpdateUserRequest
LoginRequest
ListUsersRequest
```

它们服务于 HTTP 请求，不一定等于数据库模型。

---

## 三、请求 DTO 示例

注册请求：

```go
type RegisterRequest struct {
    Username string `json:"username" binding:"required,min=2,max=32"`
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required,min=8,max=64"`
}
```

注意：

- 没有 `ID`。
- 没有 `Role`。
- 没有 `PasswordHash`。
- 有明文 `Password`，但它只存在于请求入口。

handler 读取后，应该尽快交给 service 处理，不要把明文密码到处传。

---

## 四、service input 示例

service 不一定要直接接收 request。

可以定义：

```go
type RegisterInput struct {
    Username string
    Email    string
    Password string
}
```

handler 中转换：

```go
user, err := h.authService.Register(service.RegisterInput{
    Username: req.Username,
    Email:    req.Email,
    Password: req.Password,
})
```

为什么要多一层？

因为 request 属于 HTTP 层，service input 属于业务层。

今天数据来自 HTTP，明天也可能来自命令行、消息队列、测试代码。service 不应该强依赖 HTTP DTO。

---

## 五、业务 model 示例

```go
type User struct {
    ID           uint
    Username     string
    Email        string
    PasswordHash string
    Role         string
}
```

业务 model 表示系统内部的用户对象。

它可以包含 `PasswordHash`，因为业务内部需要用它校验密码。

但对外响应时要隐藏：

```go
PasswordHash string `json:"-"`
```

---

## 六、响应 DTO

有时还需要专门定义响应结构体。

例如：

```go
type UserResponse struct {
    ID       uint   `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email"`
    Role     string `json:"role"`
}
```

转换：

```go
func ToUserResponse(user model.User) UserResponse {
    return UserResponse{
        ID:       user.ID,
        Username: user.Username,
        Email:    user.Email,
        Role:     user.Role,
    }
}
```

这样即使 model 里有敏感字段，也不会误返回。

入门阶段可以先用 `json:"-"`，项目变复杂后再引入响应 DTO。

---

## 七、列表查询 DTO

查询参数 DTO：

```go
type ListUsersRequest struct {
    Page     int    `form:"page" binding:"omitempty,gte=1"`
    PageSize int    `form:"page_size" binding:"omitempty,gte=1,lte=100"`
    Status   string `form:"status" binding:"omitempty,oneof=active disabled"`
    Keyword  string `form:"keyword" binding:"omitempty,max=50"`
}
```

service input：

```go
type ListUsersInput struct {
    Page     int
    PageSize int
    Status   string
    Keyword  string
}
```

转换：

```go
result, err := h.userService.List(service.ListUsersInput{
    Page:     req.Page,
    PageSize: req.PageSize,
    Status:   req.Status,
    Keyword:  req.Keyword,
})
```

---

## 八、什么时候可以复用结构体

小项目里，有时可以复用。

例如：

```go
type UpdateUserRequest struct {
    Username string `json:"username"`
    Email    string `json:"email"`
}
```

如果 service input 完全一样，复用不会立刻出问题。

但遇到这些情况时，建议分开：

- 请求字段和数据库字段不同。
- 有敏感字段。
- 有只读字段。
- 有服务端生成字段。
- 有不同入口调用同一业务逻辑。
- 需要清楚表达层级边界。

---

## 九、常见危险字段

不要让前端直接控制：

```text
id
user_id
role
password_hash
created_at
updated_at
deleted_at
is_admin
balance
owner_id
```

例如创建任务时，请求体不应该包含：

```json
{
  "user_id": 999
}
```

任务的 `user_id` 应该来自 JWT 认证中间件，而不是来自前端请求体。

---

## 十、DTO 放在哪个包

常见做法：

```text
handler 包里定义 request DTO。
service 包里定义 input DTO。
model 包里定义业务/数据库 model。
```

例如：

```text
internal/handler/user_handler.go      CreateUserRequest
internal/service/user_service.go      CreateUserInput
internal/model/user.go                User
```

这样边界最清楚。

---

## 十一、本节练习

为用户注册设计三种结构体：

1. `RegisterRequest`
2. `RegisterInput`
3. `User`

要求：

- `RegisterRequest` 有 `Password`。
- `User` 有 `PasswordHash`。
- 响应中不能出现 `PasswordHash`。
- `Role` 默认由 service 设置为 `user`。
- 前端不能通过请求体设置 `Role`。

---

## 十二、本节验收

你应该能够回答：

- DTO 是什么？
- 为什么 RegisterRequest 不应该包含 Role？
- 为什么 User model 可以有 PasswordHash，但响应不能返回它？
- handler request 和 service input 为什么可以分开？
- 哪些字段不应该让前端直接控制？


