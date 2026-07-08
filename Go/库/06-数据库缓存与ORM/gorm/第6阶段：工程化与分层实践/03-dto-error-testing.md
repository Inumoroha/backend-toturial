# 03 DTO、错误处理与测试

本节目标：围绕「dto error testing」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

真实项目里，不建议把 GORM Model 直接返回给前端。Model 是数据库结构，DTO 是接口结构，两者职责不同。

## 1. 为什么 Model 和 DTO 要分离

模型示例：

```go
type User struct {
    ID           uint
    Username     string
    Email        string
    PasswordHash string
    CreatedAt    time.Time
}
```

如果直接返回 `User`，可能把 `PasswordHash` 暴露给前端。

DTO：

```go
type UserDTO struct {
    ID       uint   `json:"id"`
    Username string `json:"username"`
}
```

转换函数：

```go
func ToUserDTO(user model.User) UserDTO {
    return UserDTO{
        ID:       user.ID,
        Username: user.Username,
    }
}
```

## 2. 请求 DTO

```go
type CreateArticleRequest struct {
    CategoryID uint   `json:"category_id" binding:"required"`
    Title      string `json:"title" binding:"required"`
    Content    string `json:"content" binding:"required"`
    TagIDs     []uint `json:"tag_ids"`
}
```

请求 DTO 用于接收前端输入，不要直接绑定到 GORM Model。

## 3. 响应 DTO

```go
type ArticleDTO struct {
    ID       uint      `json:"id"`
    Title    string    `json:"title"`
    Summary  string    `json:"summary"`
    Author   UserDTO   `json:"author"`
    CreatedAt time.Time `json:"created_at"`
}
```

响应 DTO 用于控制前端能看到什么。

## 4. 统一错误

可以定义业务错误：

```go
type AppError struct {
    Code    string
    Message string
}

func (e *AppError) Error() string {
    return e.Message
}
```

常见错误：

```go
var ErrNotFound = &AppError{Code: "NOT_FOUND", Message: "resource not found"}
var ErrForbidden = &AppError{Code: "FORBIDDEN", Message: "forbidden"}
var ErrInvalidInput = &AppError{Code: "INVALID_INPUT", Message: "invalid input"}
```

Handler 里统一转换为 HTTP 响应。

## 5. Repository 测试

Repository 测试建议使用独立测试数据库，或者 SQLite 临时数据库。

测试重点：

- 创建是否成功。
- 查询条件是否正确。
- 软删除是否生效。
- 关联预加载是否正确。

示例：

```go
func TestArticleRepository_Create(t *testing.T) {
    db := setupTestDB(t)
    repo := repository.NewArticleRepository(db)

    article := &model.Article{
        UserID:     1,
        CategoryID: 1,
        Title:      "test",
        Content:    "content",
    }

    err := repo.Create(context.Background(), article)
    require.NoError(t, err)
    require.NotZero(t, article.ID)
}
```

## 6. Service 测试

Service 测试重点是业务规则：

- 标题为空不能创建文章。
- 非作者不能编辑文章。
- 库存不足不能下单。
- 事务失败要回滚。

## 本节练习

- [ ] 为用户响应定义 `UserDTO`。
- [ ] 为文章创建定义请求 DTO。
- [ ] 文章详情响应不要返回 `PasswordHash`。
- [ ] 定义统一业务错误。
- [ ] 写一个 Repository 测试。
- [ ] 写一个 Service 事务回滚测试。
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
