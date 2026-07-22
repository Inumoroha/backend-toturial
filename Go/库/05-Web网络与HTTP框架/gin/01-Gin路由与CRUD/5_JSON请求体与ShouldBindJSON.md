# 5. JSON 请求体与 ShouldBindJSON

本节目标：掌握 Gin 中读取 JSON 请求体的方法，并理解请求结构体、JSON tag 和错误处理。

---

## 一、什么时候使用 JSON 请求体

创建和更新资源时，通常使用 JSON 请求体。

例如创建用户：

```http
POST /api/v1/users
Content-Type: application/json
```

请求体：

```json
{
  "username": "alice",
  "email": "alice@example.com",
  "age": 20
}
```

这类数据不适合放在查询参数里，因为字段多、结构复杂，而且可能包含敏感信息。

---

## 二、定义请求结构体

```go
type CreateUserRequest struct {
    Username string `json:"username"`
    Email    string `json:"email"`
    Age      int    `json:"age"`
}
```

`json` tag 的作用是指定 JSON 字段名。

如果没有 tag：

```go
Username string
```

默认 JSON 字段可能变成：

```json
{
  "Username": "alice"
}
```

这不符合常见 API 风格。

---

## 三、使用 ShouldBindJSON

```go
func createUser(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(400, gin.H{
            "message": "invalid json body",
        })
        return
    }

    c.JSON(201, gin.H{
        "data": req,
    })
}
```

`ShouldBindJSON` 做了两件事：

```text
读取请求体。
把 JSON 字段绑定到 Go 结构体。
```

如果 JSON 格式错误，或者字段类型不匹配，会返回错误。

---

## 四、必须设置 Content-Type

请求中建议明确写：

```http
Content-Type: application/json
```

完整测试：

```http
### create user
POST http://localhost:8080/api/v1/users
Content-Type: application/json

{
  "username": "alice",
  "email": "alice@example.com",
  "age": 20
}
```

---

## 五、常见 JSON 错误

### 1. 少逗号

错误：

```json
{
  "username": "alice"
  "email": "alice@example.com"
}
```

正确：

```json
{
  "username": "alice",
  "email": "alice@example.com"
}
```

### 2. 字段类型错误

结构体中 `Age` 是 int：

```go
Age int `json:"age"`
```

错误请求：

```json
{
  "age": "twenty"
}
```

应该传：

```json
{
  "age": 20
}
```

### 3. 字段名不一致

结构体 tag：

```go
Username string `json:"username"`
```

请求里写：

```json
{
  "user_name": "alice"
}
```

就绑定不到 `Username`。

---

## 六、ShouldBindJSON 和 BindJSON 的区别

入门阶段优先使用：

```go
c.ShouldBindJSON(&req)
```

不要优先使用：

```go
c.BindJSON(&req)
```

原因是 `BindJSON` 在失败时会自动写入响应状态，控制权较少。`ShouldBindJSON` 只返回错误，由你自己决定响应格式。

后面做统一错误响应时，这一点很重要。

---

## 七、本节练习

实现：

```text
POST /api/v1/users
```

请求：

```json
{
  "username": "alice",
  "email": "alice@example.com",
  "age": 20
}
```

要求：

- 正常创建返回 201。
- JSON 错误返回 400。
- 响应中返回创建后的用户数据。
- 在 `requests.http` 中保留一个错误 JSON 示例。

---

## 八、本节验收

你应该能够回答：

- JSON 请求体适合什么场景？
- `json` tag 有什么作用？
- `ShouldBindJSON` 做了什么？
- 为什么请求头要写 `Content-Type: application/json`？
- 为什么推荐 `ShouldBindJSON` 而不是直接用 `BindJSON`？


