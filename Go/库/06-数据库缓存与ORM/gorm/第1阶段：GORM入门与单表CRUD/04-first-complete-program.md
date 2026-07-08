# 04 第一个完整 GORM 程序

本节目标：写出一个可以从头运行的 GORM 小程序。它会完成建表、新增、查询、更新、删除、打印 SQL，并在每一步解释 GORM 代码背后的数据库行为。

前面你已经能连接数据库，也看过基础 CRUD。本节把这些内容整理成一个完整练习。

---

## 一、本节要完成什么

最终程序会做这些事：

```text
1. 连接 MySQL
2. 开启 SQL 日志
3. 定义 User 模型
4. 自动迁移 users 表
5. 清理旧测试数据
6. 新增用户
7. 根据主键查询用户
8. 根据邮箱查询用户
9. 查询用户列表
10. 更新用户
11. 删除用户
12. 处理查询不存在的错误
```

这不是为了写一个业务系统，而是为了建立 GORM 操作数据库的完整直觉。

---

## 二、安装依赖

进入练习项目：

```powershell
cd code/gorm-study
```

安装：

```powershell
go get gorm.io/gorm
go get gorm.io/driver/mysql
```

---

## 三、完整代码

创建或替换 `main.go`：

```go
package main

import (
    "errors"
    "fmt"
    "log"
    "time"

    "gorm.io/driver/mysql"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

type User struct {
    ID        uint      `gorm:"primaryKey"`
    Name      string    `gorm:"size:64;not null"`
    Email     string    `gorm:"size:128;not null;uniqueIndex"`
    Age       int       `gorm:"not null;default:18"`
    CreatedAt time.Time
    UpdatedAt time.Time
}

func main() {
    db := openDB()

    migrate(db)
    cleanup(db)

    tom := createUser(db, "Tom", "tom@example.com", 20)

    findUserByID(db, tom.ID)
    findUserByEmail(db, "tom@example.com")
    listUsers(db)
    updateUserAge(db, tom.ID, 21)
    handleNotFound(db)
    deleteUser(db, tom.ID)
}

func openDB() *gorm.DB {
    dsn := "root:root123456@tcp(127.0.0.1:3306)/gorm_study?charset=utf8mb4&parseTime=True&loc=Local"

    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Info),
    })
    if err != nil {
        log.Fatal(err)
    }

    sqlDB, err := db.DB()
    if err != nil {
        log.Fatal(err)
    }

    sqlDB.SetMaxOpenConns(20)
    sqlDB.SetMaxIdleConns(5)
    sqlDB.SetConnMaxLifetime(time.Hour)

    return db
}

func migrate(db *gorm.DB) {
    if err := db.AutoMigrate(&User{}); err != nil {
        log.Fatal(err)
    }
}

func cleanup(db *gorm.DB) {
    if err := db.Where("email LIKE ?", "%@example.com").Delete(&User{}).Error; err != nil {
        log.Fatal(err)
    }
}

func createUser(db *gorm.DB, name, email string, age int) User {
    user := User{Name: name, Email: email, Age: age}

    if err := db.Create(&user).Error; err != nil {
        log.Fatal(err)
    }

    fmt.Println("created user id:", user.ID)
    return user
}

func findUserByID(db *gorm.DB, id uint) {
    var user User

    if err := db.First(&user, id).Error; err != nil {
        log.Fatal(err)
    }

    fmt.Printf("find by id: %+v\n", user)
}

func findUserByEmail(db *gorm.DB, email string) {
    var user User

    if err := db.Where("email = ?", email).First(&user).Error; err != nil {
        log.Fatal(err)
    }

    fmt.Printf("find by email: %+v\n", user)
}

func listUsers(db *gorm.DB) {
    var users []User

    if err := db.Order("id DESC").Limit(10).Find(&users).Error; err != nil {
        log.Fatal(err)
    }

    fmt.Println("user count:", len(users))
}

func updateUserAge(db *gorm.DB, id uint, age int) {
    result := db.Model(&User{}).
        Where("id = ?", id).
        Update("age", age)

    if result.Error != nil {
        log.Fatal(result.Error)
    }

    fmt.Println("updated rows:", result.RowsAffected)
}

func handleNotFound(db *gorm.DB) {
    var user User
    err := db.Where("email = ?", "missing@example.com").First(&user).Error
    if err == nil {
        return
    }

    if errors.Is(err, gorm.ErrRecordNotFound) {
        fmt.Println("missing@example.com not found")
        return
    }

    log.Fatal(err)
}

func deleteUser(db *gorm.DB, id uint) {
    result := db.Delete(&User{}, id)
    if result.Error != nil {
        log.Fatal(result.Error)
    }

    fmt.Println("deleted rows:", result.RowsAffected)
}
```

---

## 四、运行程序

```powershell
go run .
```

你应该能看到两类输出：

一类是我们自己打印的结果：

```text
created user id: 1
find by id: ...
find by email: ...
user count: 1
updated rows: 1
missing@example.com not found
deleted rows: 1
```

另一类是 GORM logger 打印出来的 SQL。

---

## 五、逐步理解代码

### 1. `gorm.Open` 不是立即创建一堆连接

```go
db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
```

这里得到的是 GORM 的数据库操作对象。底层连接池由 Go 标准库 `database/sql` 管理。

所以你还可以取到底层连接池：

```go
sqlDB, err := db.DB()
```

然后设置连接池参数。

### 2. `AutoMigrate` 负责对齐表结构

```go
db.AutoMigrate(&User{})
```

它会根据 `User` 模型创建或调整 `users` 表。

学习阶段可以用它快速建表。生产环境要谨慎，因为数据库迁移应该可审查、可回滚、可追踪。

### 3. `Create` 会写回主键

```go
db.Create(&user)
fmt.Println(user.ID)
```

插入成功后，自增主键会写回到结构体。

### 4. `Where` 要使用占位符

```go
db.Where("email = ?", email)
```

不要写：

```go
db.Where("email = '" + email + "'")
```

后者有 SQL 注入风险。

### 5. 更新和删除要看 `RowsAffected`

```go
result := db.Model(&User{}).Where("id = ?", id).Update("age", age)
```

要同时检查：

```go
result.Error
result.RowsAffected
```

`Error` 表示 SQL 是否执行出错，`RowsAffected` 表示影响了几行。

---

## 六、打开数据库观察

程序运行到删除前，你可以临时注释掉最后的：

```go
deleteUser(db, tom.ID)
```

然后在数据库客户端执行：

```sql
SELECT id, name, email, age, created_at, updated_at
FROM users
ORDER BY id DESC;
```

你应该能看到刚刚插入的用户。

再恢复删除逻辑，重新运行后执行：

```sql
SELECT *
FROM users
WHERE email = 'tom@example.com';
```

应该查不到数据。

---

## 七、常见错误

### 1. duplicate entry

如果你反复运行程序，可能遇到邮箱唯一索引冲突。

本节代码里用了：

```go
cleanup(db)
```

它会先删除测试邮箱，避免重复插入失败。

### 2. table does not exist

说明没有执行迁移，或者连接到了错误的数据库。

检查：

```go
migrate(db)
```

也检查 DSN 里的数据库名。

### 3. record not found 被当成致命错误

查询不存在不一定是系统异常。比如登录时邮箱不存在，应该返回“用户名或密码错误”，而不是服务崩溃。

所以要区分：

```go
errors.Is(err, gorm.ErrRecordNotFound)
```

---

## 八、本节达标标准

学完本节后，你应该能够做到：

- 写出一个完整可运行的 GORM 程序。
- 知道 `AutoMigrate` 会创建或调整表结构。
- 能解释 `Create` 为什么会回填主键。
- 能使用 `Where` 安全传参。
- 能区分 `Error` 和 `RowsAffected`。
- 能处理 `ErrRecordNotFound`。
- 能通过 SQL 日志观察 GORM 生成的 SQL。
- 能在数据库客户端验证程序写入的数据。
