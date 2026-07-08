# 02 单表 CRUD 入门

本节目标：围绕「basic crud」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

CRUD 是后端开发的基本功：

- Create：新增
- Read：查询
- Update：更新
- Delete：删除

本节用一个 `User` 模型练习 GORM 最基础的单表操作。

## 1. 定义模型

```go
type User struct {
    ID    uint
    Name  string
    Email string
    Age   int
}
```

GORM 默认约定：

- 结构体名 `User` 对应表名 `users`。
- 字段 `Name` 对应列名 `name`。
- 字段 `Email` 对应列名 `email`。
- 字段 `ID` 默认是主键。

## 2. 自动迁移

```go
if err := db.AutoMigrate(&User{}); err != nil {
    log.Fatal(err)
}
```

`AutoMigrate` 会根据结构体创建或更新表结构。学习阶段可以用它快速创建表。

注意：生产环境不要完全依赖 `AutoMigrate` 管理复杂迁移，后面工程化阶段会讲迁移策略。

## 3. 新增数据

```go
user := User{
    Name:  "Tom",
    Email: "tom@example.com",
    Age:   20,
}

if err := db.Create(&user).Error; err != nil {
    log.Fatal(err)
}

fmt.Println("created user id:", user.ID)
```

执行成功后，GORM 会把自增主键写回 `user.ID`。

## 4. 根据主键查询

```go
var user User

if err := db.First(&user, 1).Error; err != nil {
    log.Fatal(err)
}

fmt.Printf("%+v\n", user)
```

对应 SQL 大致是：

```sql
SELECT * FROM users WHERE users.id = 1 ORDER BY users.id LIMIT 1;
```

## 5. 条件查询

```go
var user User

err := db.Where("email = ?", "tom@example.com").First(&user).Error
if err != nil {
    log.Fatal(err)
}
```

`?` 是占位符，GORM 会安全绑定参数。不要用字符串拼接 SQL 条件。

错误示例：

```go
db.Where("email = '" + email + "'").First(&user)
```

这种写法容易造成 SQL 注入。

## 6. 查询多条数据

```go
var users []User

if err := db.Find(&users).Error; err != nil {
    log.Fatal(err)
}
```

`Find` 会查询符合条件的所有记录。如果没有条件，就是查询全表。

学习阶段可以这样做，真实接口里要避免无条件全表查询，通常要分页。

## 7. 更新数据

### 更新单个字段

```go
err := db.Model(&User{}).
    Where("id = ?", 1).
    Update("age", 21).Error
```

### 更新多个字段

```go
err := db.Model(&User{}).
    Where("id = ?", 1).
    Updates(map[string]any{
        "name": "Tommy",
        "age":  22,
    }).Error
```

使用 `map` 更新时，零值也会被更新。使用结构体更新时，零值默认会被忽略。这个坑后面会专门讲。

## 8. 删除数据

```go
err := db.Delete(&User{}, 1).Error
```

如果模型里没有 `DeletedAt` 字段，执行的是物理删除。

如果模型里有软删除字段，执行的是软删除，即更新 `deleted_at`。

## 9. 完整示例

```go
package main

import (
    "fmt"
    "log"

    "gorm.io/driver/mysql"
    "gorm.io/gorm"
)

type User struct {
    ID    uint
    Name  string
    Email string
    Age   int
}

func main() {
    dsn := "root:password@tcp(127.0.0.1:3306)/gorm_study?charset=utf8mb4&parseTime=True&loc=Local"
    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatal(err)
    }

    if err := db.AutoMigrate(&User{}); err != nil {
        log.Fatal(err)
    }

    user := User{Name: "Tom", Email: "tom@example.com", Age: 20}
    if err := db.Create(&user).Error; err != nil {
        log.Fatal(err)
    }

    var found User
    if err := db.First(&found, user.ID).Error; err != nil {
        log.Fatal(err)
    }
    fmt.Printf("found: %+v\n", found)

    if err := db.Model(&found).Update("age", 21).Error; err != nil {
        log.Fatal(err)
    }

    var users []User
    if err := db.Find(&users).Error; err != nil {
        log.Fatal(err)
    }
    fmt.Println("user count:", len(users))

    if err := db.Delete(&found).Error; err != nil {
        log.Fatal(err)
    }
}
```

## 本节练习

- [ ] 创建 `User` 模型。
- [ ] 使用 `AutoMigrate` 创建表。
- [ ] 新增 3 个用户。
- [ ] 根据 ID 查询用户。
- [ ] 根据邮箱查询用户。
- [ ] 查询所有用户。
- [ ] 修改用户年龄。
- [ ] 删除一个用户。

## 常见坑

- 忘记检查 `.Error`。
- `Find` 没有加条件和分页。
- 使用字符串拼接查询条件。
- 误以为 `AutoMigrate` 可以替代生产迁移。
- 不看生成的 SQL，只看 Go 代码。
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
