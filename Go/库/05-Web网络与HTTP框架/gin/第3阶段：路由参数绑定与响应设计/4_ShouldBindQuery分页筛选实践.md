# 4. ShouldBindQuery 分页筛选实践

本节目标：使用 `ShouldBindQuery` 实现列表接口中的分页、状态筛选和关键词搜索。完成后，你应该能写出一个结构清晰、参数可校验、响应格式稳定的列表接口。

列表接口是后端项目里最常见的接口类型之一。用户列表、任务列表、文章列表、订单列表，本质上都会遇到这些问题：

```text
第几页？
每页多少条？
按什么状态筛选？
是否按关键词搜索？
返回总数是多少？
```

---

## 一、目标接口

本节要实现：

```http
GET /api/v1/users?page=1&page_size=10&status=active&keyword=alice
```

参数含义：

| 参数 | 含义 | 示例 |
| --- | --- | --- |
| `page` | 当前页码 | `1` |
| `page_size` | 每页数量 | `10` |
| `status` | 用户状态 | `active` |
| `keyword` | 用户名或邮箱关键词 | `alice` |

这些参数都属于查询条件，所以放在 query string 中，而不是 JSON body。

---

## 二、为什么 GET 列表不使用 JSON body

不推荐：

```http
GET /api/v1/users
Content-Type: application/json

{
  "page": 1,
  "page_size": 10
}
```

原因：

- GET 请求通常不依赖请求体。
- 浏览器、代理、网关对 GET body 支持并不统一。
- 查询条件放在 URL 中更方便复制、调试和缓存。

推荐：

```http
GET /api/v1/users?page=1&page_size=10
```

---

## 三、定义查询参数结构体

```go
type ListUsersRequest struct {
    Page     int    `form:"page" binding:"omitempty,gte=1"`
    PageSize int    `form:"page_size" binding:"omitempty,gte=1,lte=100"`
    Status   string `form:"status" binding:"omitempty,oneof=active disabled"`
    Keyword  string `form:"keyword" binding:"omitempty,max=50"`
}
```

逐项解释：

```text
Page
可以不传；如果传，必须大于等于 1。

PageSize
可以不传；如果传，必须在 1 到 100 之间。

Status
可以不传；如果传，只能是 active 或 disabled。

Keyword
可以不传；如果传，最长 50 个字符。
```

查询参数必须使用 `form` tag：

```go
form:"page"
```

不要写成：

```go
json:"page"
```

---

## 四、绑定查询参数

handler 示例：

```go
func listUsers(c *gin.Context) {
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

    if req.Page == 0 {
        req.Page = 1
    }
    if req.PageSize == 0 {
        req.PageSize = 10
    }

    response.OK(c, req)
}
```

这里先直接返回 `req`，用于确认参数是否绑定成功。

测试：

```http
GET http://localhost:8080/api/v1/users?page=2&page_size=20&status=active&keyword=alice
```

期望响应中的 data 类似：

```json
{
  "page": 2,
  "page_size": 20,
  "status": "active",
  "keyword": "alice"
}
```

---

## 五、为什么后端要设置默认值

如果请求：

```http
GET /api/v1/users
```

没有传 `page` 和 `page_size`，后端应该给默认值：

```text
page = 1
page_size = 10
```

不要完全依赖前端传参。后端接口应该自己具备稳定行为。

常见默认值：

```go
if req.Page == 0 {
    req.Page = 1
}
if req.PageSize == 0 {
    req.PageSize = 10
}
```

---

## 六、为什么 page_size 要有上限

如果没有上限，调用方可能请求：

```http
GET /api/v1/users?page=1&page_size=1000000
```

这会带来：

- 数据库压力过大。
- 内存占用过高。
- 响应变慢。
- 影响其他用户请求。

所以即使前端正常传参，后端仍然要设置上限。

本教程使用：

```text
page_size 最大 100
```

---

## 七、内存版筛选实现

假设用户结构：

```go
type User struct {
    ID       uint   `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email"`
    Age      int    `json:"age"`
    Status   string `json:"status"`
}
```

内存数据：

```go
var users = map[uint]User{
    1: {ID: 1, Username: "alice", Email: "alice@example.com", Age: 20, Status: "active"},
    2: {ID: 2, Username: "bob", Email: "bob@example.com", Age: 22, Status: "disabled"},
    3: {ID: 3, Username: "cindy", Email: "cindy@example.com", Age: 21, Status: "active"},
}
```

筛选函数：

```go
func filterUsers(req ListUsersRequest) []User {
    result := make([]User, 0)

    for _, user := range users {
        if req.Status != "" && user.Status != req.Status {
            continue
        }

        if req.Keyword != "" {
            keyword := strings.ToLower(req.Keyword)
            username := strings.ToLower(user.Username)
            email := strings.ToLower(user.Email)

            if !strings.Contains(username, keyword) && !strings.Contains(email, keyword) {
                continue
            }
        }

        result = append(result, user)
    }

    return result
}
```

需要导入：

```go
import "strings"
```

---

## 八、内存版分页实现

分页计算：

```go
func paginateUsers(items []User, page, pageSize int) []User {
    start := (page - 1) * pageSize
    if start >= len(items) {
        return []User{}
    }

    end := start + pageSize
    if end > len(items) {
        end = len(items)
    }

    return items[start:end]
}
```

示例：

```text
总共 25 条
page = 2
page_size = 10
start = 10
end = 20
```

返回第 11 到第 20 条。

注意：Go 切片下标从 0 开始。

---

## 九、完整 listUsers 示例

```go
func listUsers(c *gin.Context) {
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

    if req.Page == 0 {
        req.Page = 1
    }
    if req.PageSize == 0 {
        req.PageSize = 10
    }

    filtered := filterUsers(req)
    paged := paginateUsers(filtered, req.Page, req.PageSize)

    response.OK(c, response.PageResult{
        Items:    paged,
        Total:    int64(len(filtered)),
        Page:     req.Page,
        PageSize: req.PageSize,
    })
}
```

响应示例：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "items": [
      {
        "id": 1,
        "username": "alice",
        "email": "alice@example.com",
        "age": 20,
        "status": "active"
      }
    ],
    "total": 1,
    "page": 1,
    "page_size": 10
  }
}
```

---

## 十、常见测试请求

正常请求：

```http
GET http://localhost:8080/api/v1/users?page=1&page_size=10
```

按状态筛选：

```http
GET http://localhost:8080/api/v1/users?status=active
```

关键词搜索：

```http
GET http://localhost:8080/api/v1/users?keyword=alice
```

非法页码：

```http
GET http://localhost:8080/api/v1/users?page=0
```

非法每页数量：

```http
GET http://localhost:8080/api/v1/users?page_size=1000
```

非法状态：

```http
GET http://localhost:8080/api/v1/users?status=unknown
```

---

## 十一、常见问题

### 1. 查询参数绑定后都是 0

检查：

```go
form:"page"
```

是否误写成：

```go
json:"page"
```

### 2. page_size 不传时校验失败

检查是否写了 `omitempty`：

```go
binding:"omitempty,gte=1,lte=100"
```

如果没有 `omitempty`，空值也会进入后续校验。

### 3. 分页结果为空

检查：

- page 是否超出范围。
- 筛选条件是否过窄。
- `start >= len(items)` 时是否返回空切片。

---

## 十二、本节练习

完成：

1. 给用户增加 `Status` 字段。
2. 实现 `ListUsersRequest`。
3. 使用 `ShouldBindQuery` 绑定分页筛选参数。
4. 实现状态筛选。
5. 实现关键词搜索。
6. 实现内存分页。
7. 使用 `PageResult` 返回分页结果。
8. 写 6 个 `requests.http` 测试请求。

---

## 十三、本节验收

你应该能够回答：

- 为什么列表接口使用 query 参数？
- `ShouldBindQuery` 和 `ShouldBindJSON` 的区别是什么？
- `omitempty` 在分页参数中有什么作用？
- 为什么 `page_size` 必须限制最大值？
- 内存分页中的 `start` 和 `end` 如何计算？
- 为什么分页响应需要 `total`？

