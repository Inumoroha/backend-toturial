# 2. binding 校验与字段级错误

本节目标：使用 Gin 的 binding tag 校验请求参数，并把校验失败转换成更容易调试的字段级错误响应。

上一阶段你已经会写：

```go
var req CreateUserRequest
if err := c.ShouldBindJSON(&req); err != nil {
    c.JSON(400, gin.H{"message": "invalid request"})
    return
}
```

这能挡住错误请求，但还不够细。真实开发中你至少要知道：

- 是 JSON 格式错误。
- 是字段缺失。
- 是邮箱格式不对。
- 是数字范围不对。
- 是枚举值不合法。

---

## 一、为什么需要参数校验

假设创建用户接口允许下面请求通过：

```json
{
  "username": "",
  "email": "not-email",
  "age": 200,
  "status": "unknown"
}
```

这会带来几个问题：

- 数据库中出现脏数据。
- service 层要处理大量本该在入口拒绝的非法输入。
- 前端不知道自己传错了什么。
- 后续业务逻辑更容易出现异常。

所以 handler 的一个重要职责是：

```text
把明显非法的 HTTP 请求挡在入口处。
```

---

## 二、创建用户请求结构体

定义：

```go
type CreateUserRequest struct {
    Username string `json:"username" binding:"required,min=2,max=32"`
    Email    string `json:"email" binding:"required,email"`
    Age      int    `json:"age" binding:"gte=1,lte=120"`
    Status   string `json:"status" binding:"omitempty,oneof=active disabled"`
}
```

逐项解释：

```text
Username
必须传，长度 2 到 32。

Email
必须传，并且必须符合邮箱格式。

Age
必须是 1 到 120 之间的数字。

Status
可以不传；如果传，只能是 active 或 disabled。
```

---

## 三、常用 binding 规则

| 规则 | 含义 | 示例 |
| --- | --- | --- |
| `required` | 必填 | `binding:"required"` |
| `min` | 最小长度或最小值 | `binding:"min=2"` |
| `max` | 最大长度或最大值 | `binding:"max=32"` |
| `email` | 邮箱格式 | `binding:"email"` |
| `gte` | 大于等于 | `binding:"gte=1"` |
| `lte` | 小于等于 | `binding:"lte=120"` |
| `oneof` | 枚举值 | `binding:"oneof=active disabled"` |
| `omitempty` | 空值时跳过后续校验 | `binding:"omitempty,oneof=active disabled"` |

注意：

```go
binding:"omitempty,oneof=active disabled"
```

表示字段为空时不校验。如果字段非空，就必须是 `active` 或 `disabled`。

---

## 四、ShouldBindJSON

handler 示例：

```go
func createUser(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        response.Fail(c, http.StatusBadRequest, response.CodeInvalidRequest, "invalid request")
        return
    }

    response.Created(c, req)
}
```

`ShouldBindJSON` 会：

```text
读取请求体。
解析 JSON。
把 JSON 字段绑定到结构体。
执行 binding 校验。
```

失败时返回 error，但不会自动写响应。这给了你统一响应的控制权。

---

## 五、错误 JSON 示例

### 1. JSON 格式错误

错误请求：

```json
{
  "username": "alice"
  "email": "alice@example.com"
}
```

少了逗号，JSON 解析失败。

### 2. 字段类型错误

错误请求：

```json
{
  "username": "alice",
  "email": "alice@example.com",
  "age": "twenty"
}
```

`age` 应该是数字，不是字符串。

### 3. 校验规则失败

错误请求：

```json
{
  "username": "a",
  "email": "abc",
  "age": 200,
  "status": "unknown"
}
```

JSON 格式没问题，但字段校验失败。

---

## 六、字段级错误响应

只返回：

```json
{
  "code": 40001,
  "message": "invalid request"
}
```

可以用于生产环境，但学习和调试阶段更希望看到字段错误：

```json
{
  "code": 40001,
  "message": "invalid request",
  "data": {
    "Username": "min",
    "Email": "email",
    "Age": "lte",
    "Status": "oneof"
  }
}
```

这能快速定位问题。

---

## 七、实现 FormatValidationError

创建：

```text
internal/response/validation.go
```

代码：

```go
package response

import (
    "errors"

    "github.com/go-playground/validator/v10"
)

func FormatValidationError(err error) map[string]string {
    result := make(map[string]string)

    var validationErrors validator.ValidationErrors
    if errors.As(err, &validationErrors) {
        for _, fieldError := range validationErrors {
            result[fieldError.Field()] = fieldError.Tag()
        }
        return result
    }

    result["body"] = "invalid json"
    return result
}
```

这里用了：

```go
errors.As(err, &validationErrors)
```

它的作用是判断这个错误里是否包含 `validator.ValidationErrors` 类型。

---

## 八、增加 FailWithData

统一响应中增加：

```go
func FailWithData(c *gin.Context, httpStatus int, code int, message string, data interface{}) {
    c.JSON(httpStatus, Body{
        Code:    code,
        Message: message,
        Data:    data,
    })
}
```

handler 中使用：

```go
if err := c.ShouldBindJSON(&req); err != nil {
    response.FailWithData(
        c,
        http.StatusBadRequest,
        response.CodeInvalidRequest,
        "invalid request",
        response.FormatValidationError(err),
    )
    return
}
```

---

## 九、字段名为什么是大写

上面的错误响应可能是：

```json
{
  "Username": "required"
}
```

而不是：

```json
{
  "username": "required"
}
```

原因是 `fieldError.Field()` 返回的是 Go 结构体字段名。

学习阶段先接受这个结果。进阶时可以通过反射读取 `json` tag，把 Go 字段名转换为 JSON 字段名。

更友好的目标是：

```json
{
  "username": "required",
  "email": "email"
}
```

---

## 十、查询参数绑定

列表接口请求：

```http
GET /api/v1/users?page=1&page_size=10&status=active
```

结构体：

```go
type ListUsersRequest struct {
    Page     int    `form:"page" binding:"omitempty,gte=1"`
    PageSize int    `form:"page_size" binding:"omitempty,gte=1,lte=100"`
    Status   string `form:"status" binding:"omitempty,oneof=active disabled"`
}
```

绑定：

```go
var req ListUsersRequest
if err := c.ShouldBindQuery(&req); err != nil {
    response.FailWithData(
        c,
        http.StatusBadRequest,
        response.CodeInvalidRequest,
        "invalid query",
        response.FormatValidationError(err),
    )
    return
}
```

注意：查询参数使用 `form` tag，不是 `json` tag。

---

## 十一、常见问题

### 1. 校验没有生效

检查：

- 是否使用了 `binding` tag。
- 是否调用了 `ShouldBindJSON` 或 `ShouldBindQuery`。
- 请求 Content-Type 是否正确。

### 2. 查询参数绑定不到

检查结构体 tag 是否写成了：

```go
form:"page"
```

而不是：

```go
json:"page"
```

### 3. 数字字段传空字符串

例如：

```text
?page=
```

可能导致绑定失败。前端应避免传空字符串，后端也要做好错误响应。

---

## 十二、本节练习

完成：

1. 给创建用户接口增加 `binding` 校验。
2. 给更新用户接口增加 `binding` 校验。
3. 给列表接口增加 `ShouldBindQuery` 校验。
4. 增加 `FailWithData`。
5. 增加 `FormatValidationError`。
6. 在 `requests.http` 中保留至少 3 个错误请求。

---

## 十三、本节验收

你应该能够回答：

- `binding:"required"` 解决什么问题？
- `omitempty` 有什么作用？
- `ShouldBindJSON` 和 `ShouldBindQuery` 的区别是什么？
- 为什么查询参数结构体使用 `form` tag？
- `errors.As` 在格式化校验错误时有什么作用？
- 为什么生产环境不一定要返回非常详细的内部错误？

