# 05-01 数据库、GORM 与 Repository

本阶段目标：把前面内存 map 中的数据改成数据库存储，并理解 Gin 项目中数据库层应该如何组织。

## 一、为什么要引入数据库

内存 map 适合学习路由和分层，但有明显问题：

- 服务重启后数据丢失。
- 多实例部署时数据不共享。
- 并发读写需要额外保护。
- 无法方便做查询、索引、事务和统计。

真实后端服务通常使用 MySQL、PostgreSQL 等关系型数据库保存核心业务数据。

本教程以 MySQL 为例，PostgreSQL 思路类似。

## 二、安装依赖

```bash
go get gorm.io/gorm
go get gorm.io/driver/mysql
```

如果使用 PostgreSQL：

```bash
go get gorm.io/driver/postgres
```

## 三、准备数据库

创建数据库：

```sql
CREATE DATABASE gin_learning DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

建议创建独立用户，不要在生产环境直接使用 root 用户。

开发阶段可以先使用：

```text
用户名：root
密码：你的本地密码
数据库：gin_learning
```

## 四、建立数据库连接

新建：

```text
internal/database/database.go
```

示例：

```go
package database

import (
    "fmt"
    "log"
    "time"

    "gorm.io/driver/mysql"
    "gorm.io/gorm"
)

func NewMySQL() *gorm.DB {
    dsn := "root:password@tcp(127.0.0.1:3306)/gin_learning?charset=utf8mb4&parseTime=True&loc=Local"

    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatalf("connect database failed: %v", err)
    }

    sqlDB, err := db.DB()
    if err != nil {
        log.Fatalf("get sql db failed: %v", err)
    }

    sqlDB.SetMaxIdleConns(10)
    sqlDB.SetMaxOpenConns(100)
    sqlDB.SetConnMaxLifetime(time.Hour)

    fmt.Println("database connected")
    return db
}
```

后面会把 DSN 放进配置文件。现在先写死，重点是理解连接流程。

## 五、定义 GORM 模型

修改 `internal/model/user.go`：

```go
package model

import "time"

type User struct {
    ID           uint           `json:"id" gorm:"primaryKey"`
    Username     string         `json:"username" gorm:"size:32;not null"`
    Email        string         `json:"email" gorm:"size:128;not null;uniqueIndex"`
    PasswordHash string         `json:"-" gorm:"size:255;not null"`
    Age          int            `json:"age"`
    Status       string         `json:"status" gorm:"size:20;not null;default:active"`
    CreatedAt    time.Time      `json:"created_at"`
    UpdatedAt    time.Time      `json:"updated_at"`
    DeletedAt    gorm.DeletedAt `json:"-" gorm:"index"`
}
```

注意：

- `json:"-"` 表示不返回给前端。
- `uniqueIndex` 表示唯一索引。
- `gorm.DeletedAt` 表示软删除。

如果使用了 `gorm.DeletedAt`，需要导入：

```go
import "gorm.io/gorm"
```

## 六、自动迁移

在启动时执行：

```go
db.AutoMigrate(&model.User{})
```

示例 main：

```go
db := database.NewMySQL()
if err := db.AutoMigrate(&model.User{}); err != nil {
    log.Fatal(err)
}
```

`AutoMigrate` 适合学习和早期开发。生产环境更推荐使用数据库迁移工具，例如 goose、golang-migrate。

## 七、改造 Repository

`internal/repository/user_repository.go`

```go
package repository

import (
    "errors"

    "gin-learning/internal/model"

    "gorm.io/gorm"
)

var ErrUserNotFound = errors.New("user not found")

type UserRepository struct {
    db *gorm.DB
}

func NewUserRepository(db *gorm.DB) *UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) Create(user *model.User) error {
    return r.db.Create(user).Error
}

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

func (r *UserRepository) List(page, pageSize int, status string) ([]model.User, int64, error) {
    var users []model.User
    var total int64

    query := r.db.Model(&model.User{})
    if status != "" {
        query = query.Where("status = ?", status)
    }

    if err := query.Count(&total).Error; err != nil {
        return nil, 0, err
    }

    offset := (page - 1) * pageSize
    err := query.Order("id DESC").Offset(offset).Limit(pageSize).Find(&users).Error
    if err != nil {
        return nil, 0, err
    }

    return users, total, nil
}

func (r *UserRepository) Update(user *model.User) error {
    return r.db.Save(user).Error
}

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

## 八、分页响应

分页接口建议返回：

```json
{
  "code": 0,
  "message": "ok",
  "data": {
    "items": [],
    "total": 100,
    "page": 1,
    "page_size": 10
  }
}
```

定义结构：

```go
type PageResult struct {
    Items    interface{} `json:"items"`
    Total    int64       `json:"total"`
    Page     int         `json:"page"`
    PageSize int         `json:"page_size"`
}
```

## 九、事务

当多个数据库操作必须一起成功或一起失败时，需要事务。

示例：

```go
err := db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&user).Error; err != nil {
        return err
    }

    if err := tx.Create(&profile).Error; err != nil {
        return err
    }

    return nil
})
```

只要函数返回错误，事务就会回滚。

## 十、常见问题

### 1. `parseTime=True` 很重要

MySQL DSN 中建议加：

```text
parseTime=True&loc=Local
```

否则时间字段可能扫描失败或时区不符合预期。

### 2. 不要在 handler 里直接使用 `db`

handler 直接查数据库会让 HTTP 和数据访问耦合。后面业务复杂时会很难测试。

### 3. 唯一索引错误要单独处理

例如邮箱重复时，不要返回笼统的 `500`，应该返回 `409 Conflict`。

## 十一、阶段练习

完成数据库版用户模块：

- 使用 GORM 替换内存 map。
- 用户邮箱唯一。
- 列表接口支持分页。
- 列表接口支持状态筛选。
- 删除用户使用软删除。
- 创建用户时如果邮箱重复，返回 `40901`。

## 十二、验收清单

你应该能够回答：

- GORM 模型和普通结构体有什么区别？
- `AutoMigrate` 适合什么场景？
- repository 为什么接收 `*gorm.DB`？
- 分页查询为什么要同时查 `items` 和 `total`？
- 什么场景需要事务？

