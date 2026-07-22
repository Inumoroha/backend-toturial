# 4. Repository 设计实践

本节目标：把 GORM 操作封装到 repository 中，让 handler 和 service 不直接依赖数据库细节。完成后，你应该能把第5阶段的内存 repository 替换成 GORM repository。

---

## 一、为什么要封装 repository

不推荐在 handler 中写：

```go
func (h *UserHandler) Get(c *gin.Context) {
    var user model.User
    err := h.db.First(&user, c.Param("id")).Error
    ...
}
```

问题：

- handler 同时处理 HTTP 和数据库。
- 难以测试。
- 数据库错误散落在各个 handler。
- 将来换存储方式影响大。

推荐：

```text
handler -> service -> repository -> GORM
```

---

## 二、UserRepository 结构

```go
package repository

import (
    "errors"

    "gin-layer-demo/internal/model"

    "gorm.io/gorm"
)

var ErrUserNotFound = errors.New("user not found")
var ErrEmailAlreadyExists = errors.New("email already exists")

type UserRepository struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) *UserRepository {
    return &UserRepository{db: db}
}
```

repository 持有 `*gorm.DB`。

main 中创建：

```go
userRepo := repository.NewUserRepository(db)
```

---

## 三、Create

```go
func (r *UserRepository) Create(user *model.User) error {
    err := r.db.Create(user).Error
    if isDuplicateEntryError(err) {
        return ErrEmailAlreadyExists
    }
    return err
}
```

调用后，`user.ID` 会被 GORM 写入数据库生成的 ID。

---

## 四、GetByID

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

注意：

```text
查不到 -> ErrUserNotFound
数据库异常 -> 原样返回
```

---

## 五、GetByEmail

```go
func (r *UserRepository) GetByEmail(email string) (*model.User, error) {
    var user model.User
    err := r.db.Where("email = ?", email).First(&user).Error
    if errors.Is(err, gorm.ErrRecordNotFound) {
        return nil, ErrUserNotFound
    }
    if err != nil {
        return nil, err
    }
    return &user, nil
}
```

邮箱查询常用于：

- 注册前检查重复。
- 登录时查用户。
- 修改邮箱时检查冲突。

---

## 六、Update

```go
func (r *UserRepository) Update(user *model.User) error {
    err := r.db.Save(user).Error
    if isDuplicateEntryError(err) {
        return ErrEmailAlreadyExists
    }
    return err
}
```

使用 `Save` 前，通常先查询出原记录，再修改字段。

这样可以避免误覆盖不该更新的字段。

---

## 七、Delete

```go
func (r *UserRepository) Delete(id uint) error {
    result := r.db.Delete(&model.User{}, id)
    if result.Error != nil {
        return result.Error
    }
    if result.RowsAffected == 0 {
        return ErrUserNotFound
    }
    return nil
}
```

`RowsAffected == 0` 表示没有记录被删除。

如果模型有 `gorm.DeletedAt`，这里执行软删除。

---

## 八、List

```go
type ListUsersParams struct {
    Page     int
    PageSize int
    Status   string
    Keyword  string
}

func (r *UserRepository) List(params ListUsersParams) ([]model.User, int64, error) {
    var users []model.User
    var total int64

    query := r.db.Model(&model.User{})

    if params.Status != "" {
        query = query.Where("status = ?", params.Status)
    }

    if params.Keyword != "" {
        like := "%" + params.Keyword + "%"
        query = query.Where("username LIKE ? OR email LIKE ?", like, like)
    }

    if err := query.Count(&total).Error; err != nil {
        return nil, 0, err
    }

    offset := (params.Page - 1) * params.PageSize
    err := query.Order("id DESC").Offset(offset).Limit(params.PageSize).Find(&users).Error
    if err != nil {
        return nil, 0, err
    }

    return users, total, nil
}
```

---

## 九、为什么 List 参数用结构体

如果参数越来越多：

```go
List(page int, pageSize int, status string, keyword string, sort string, startTime time.Time, endTime time.Time)
```

函数会很难维护。

使用结构体更清晰：

```go
List(ListUsersParams{...})
```

---

## 十、repository 不应该做什么

repository 不应该：

- 返回 HTTP 状态码。
- 调用 `response.Fail`。
- 读取 `gin.Context`。
- 决定接口文案。

错误示例：

```go
func (r *UserRepository) GetByID(c *gin.Context) {}
```

repository 应该和 Gin 无关。

---

## 十一、本节练习

完成 GORM 版 `UserRepository`：

- `Create`
- `GetByID`
- `GetByEmail`
- `Update`
- `Delete`
- `List`

要求：

- 查不到返回 `ErrUserNotFound`。
- 邮箱重复返回 `ErrEmailAlreadyExists`。
- 列表返回 `items` 和 `total`。
- repository 不导入 Gin。

---

## 十二、本节验收

你应该能够回答：

- repository 为什么持有 `*gorm.DB`？
- `RowsAffected == 0` 能说明什么？
- 为什么 List 参数适合用结构体？
- repository 为什么不应该返回 HTTP 状态码？
- GORM 版 repository 和内存版 repository 对 service 来说应该有什么相似之处？


