# 2. GORM 模型、CRUD、分页与软删除

本节目标：定义 GORM 模型，并使用 GORM 完成创建、查询、更新、删除、分页和软删除。完成本节后，你应该能把一个简单的用户表映射为 Go 结构体，并用 GORM 操作它。

---

## 一、GORM 模型是什么

GORM 模型本质上是 Go 结构体，但它通过 tag 描述数据库表结构。

例如：

```go
type User struct {
    ID       uint   `gorm:"primaryKey" json:"id"`
    Username string `gorm:"size:32;not null" json:"username"`
    Email    string `gorm:"size:128;not null;uniqueIndex" json:"email"`
}
```

这里有两类 tag：

```text
gorm tag
告诉 GORM 如何映射数据库。

json tag
告诉 Gin 返回 JSON 时字段名是什么。
```

---

## 二、定义 User 模型

创建：

```text
internal/model/user.go
```

代码：

```go
package model

import (
    "time"

    "gorm.io/gorm"
)

type User struct {
    ID        uint           `json:"id" gorm:"primaryKey"`
    Username  string         `json:"username" gorm:"size:32;not null"`
    Email     string         `json:"email" gorm:"size:128;not null;uniqueIndex"`
    Age       int            `json:"age"`
    Status    string         `json:"status" gorm:"size:20;not null;default:active;index"`
    CreatedAt time.Time      `json:"created_at"`
    UpdatedAt time.Time      `json:"updated_at"`
    DeletedAt gorm.DeletedAt `json:"-" gorm:"index"`
}
```

字段说明：

- `ID`：主键。
- `Username`：用户名，不能为空。
- `Email`：邮箱，不能为空且唯一。
- `Status`：状态，默认 active。
- `CreatedAt`：创建时间，GORM 自动维护。
- `UpdatedAt`：更新时间，GORM 自动维护。
- `DeletedAt`：软删除字段。

---

## 三、常用 gorm tag

| tag | 含义 |
| --- | --- |
| `primaryKey` | 主键 |
| `size:32` | 字符串长度 |
| `not null` | 非空 |
| `uniqueIndex` | 唯一索引 |
| `index` | 普通索引 |
| `default:active` | 默认值 |

注意：GORM tag 不是请求校验。请求校验仍然使用 Gin binding。

---

## 四、AutoMigrate

在 main 中执行：

```go
if err := db.AutoMigrate(&model.User{}); err != nil {
    log.Fatal(err)
}
```

作用：

```text
根据模型自动创建或更新表结构。
```

学习阶段很方便。

但生产环境更推荐使用迁移工具，例如 goose 或 golang-migrate，因为生产数据库变更需要更可控。

---

## 五、Create：创建记录

```go
user := model.User{
    Username: "alice",
    Email:    "alice@example.com",
    Age:      20,
    Status:   "active",
}

if err := db.Create(&user).Error; err != nil {
    return err
}
```

创建成功后，GORM 会把数据库生成的 ID 写回 `user.ID`。

---

## 六、First：查询单条记录

按主键查询：

```go
var user model.User
err := db.First(&user, 1).Error
```

按条件查询：

```go
err := db.Where("email = ?", "alice@example.com").First(&user).Error
```

如果查不到，GORM 返回：

```go
gorm.ErrRecordNotFound
```

应该单独处理，而不是直接当成 500。

---

## 七、Find：查询多条记录

```go
var users []model.User
err := db.Where("status = ?", "active").Find(&users).Error
```

如果没有数据，`Find` 通常不会返回 `ErrRecordNotFound`，而是得到空切片。

这和 `First` 不一样。

---

## 八、Save 和 Updates：更新记录

先查询，再修改，再保存：

```go
var user model.User
if err := db.First(&user, id).Error; err != nil {
    return err
}

user.Username = "alice-new"
user.Status = "disabled"

if err := db.Save(&user).Error; err != nil {
    return err
}
```

也可以使用 Updates：

```go
err := db.Model(&model.User{}).
    Where("id = ?", id).
    Updates(map[string]interface{}{
        "username": "alice-new",
        "status": "disabled",
    }).Error
```

入门阶段先掌握 `Save`。

---

## 九、Delete：删除记录

```go
err := db.Delete(&model.User{}, id).Error
```

因为模型里有：

```go
DeletedAt gorm.DeletedAt
```

GORM 默认执行软删除。

也就是说，它不会真正删除数据，而是更新 `deleted_at` 字段。

后续普通查询默认不会查出软删除的数据。

---

## 十、硬删除

如果真的要物理删除：

```go
db.Unscoped().Delete(&model.User{}, id)
```

学习阶段知道即可。真实项目中要非常谨慎。

---

## 十一、分页查询

分页公式：

```go
offset := (page - 1) * pageSize
```

查询：

```go
var users []model.User
err := db.Order("id DESC").
    Offset(offset).
    Limit(pageSize).
    Find(&users).Error
```

查总数：

```go
var total int64
err := db.Model(&model.User{}).Count(&total).Error
```

列表接口通常需要同时返回：

```text
items
total
page
page_size
```

---

## 十二、筛选和分页组合

```go
query := db.Model(&model.User{})

if status != "" {
    query = query.Where("status = ?", status)
}

if keyword != "" {
    like := "%" + keyword + "%"
    query = query.Where("username LIKE ? OR email LIKE ?", like, like)
}

var total int64
if err := query.Count(&total).Error; err != nil {
    return nil, 0, err
}

var users []model.User
offset := (page - 1) * pageSize
if err := query.Order("id DESC").Offset(offset).Limit(pageSize).Find(&users).Error; err != nil {
    return nil, 0, err
}
```

注意：这里的 `query` 会不断追加条件。

---

## 十三、常见问题

### 1. AutoMigrate 没创建表

检查：

- 是否真的执行了 `db.AutoMigrate(&model.User{})`。
- 是否连接到了正确数据库。
- 程序启动是否报错。

### 2. DeletedAt 编译错误

检查是否导入：

```go
import "gorm.io/gorm"
```

### 3. 查询不到刚删除的数据

如果是软删除，这是正常的。普通查询默认过滤 `deleted_at` 不为空的数据。

### 4. 字段名和数据库列名不一致

GORM 默认会把 `CreatedAt` 映射为 `created_at`。

---

## 十四、本节练习

完成：

1. 定义 `model.User`。
2. 执行 `AutoMigrate`。
3. 创建一条用户记录。
4. 根据 ID 查询用户。
5. 根据邮箱查询用户。
6. 更新用户状态。
7. 删除用户并观察软删除效果。
8. 实现分页查询。

---

## 十五、本节验收

你应该能够回答：

- GORM tag 和 json tag 分别有什么作用？
- `AutoMigrate` 适合什么阶段？
- `First` 和 `Find` 查不到数据时行为有什么区别？
- `DeletedAt` 如何实现软删除？
- 分页查询为什么要同时查 `items` 和 `total`？

