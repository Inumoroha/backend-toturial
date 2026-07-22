# 3. 数据表设计与 GORM 模型

本节目标：设计 users 和 tasks 两张表，并用 GORM 模型表达它们。

---

## 一、安装 GORM

以 MySQL 为例：

```bash
go get gorm.io/gorm
go get gorm.io/driver/mysql
```

如果你使用 PostgreSQL：

```bash
go get gorm.io/driver/postgres
```

---

## 二、users 表设计

用户表字段：

```text
id
username
email
password_hash
role
created_at
updated_at
deleted_at
```

设计要点：

- `email` 必须唯一。
- `password_hash` 不能返回给前端。
- `role` 用于后续权限扩展。
- `deleted_at` 用于软删除。

GORM 模型 `internal/model/user.go`：

```go
package model

import (
    "time"

    "gorm.io/gorm"
)

type User struct {
    ID           uint           `json:"id" gorm:"primaryKey"`
    Username     string         `json:"username" gorm:"size:32;not null"`
    Email        string         `json:"email" gorm:"size:128;not null;uniqueIndex"`
    PasswordHash string         `json:"-" gorm:"size:255;not null"`
    Role         string         `json:"role" gorm:"size:20;not null;default:user"`
    CreatedAt    time.Time      `json:"created_at"`
    UpdatedAt    time.Time      `json:"updated_at"`
    DeletedAt    gorm.DeletedAt `json:"-" gorm:"index"`
}
```

---

## 三、tasks 表设计

任务表字段：

```text
id
user_id
title
description
status
created_at
updated_at
deleted_at
```

设计要点：

- `user_id` 表示任务属于哪个用户。
- 查询任务时必须带 `user_id`。
- `status` 限制为 `todo`、`doing`、`done`。
- `deleted_at` 用于软删除。

GORM 模型 `internal/model/task.go`：

```go
package model

import (
    "time"

    "gorm.io/gorm"
)

type Task struct {
    ID          uint           `json:"id" gorm:"primaryKey"`
    UserID      uint           `json:"user_id" gorm:"not null;index"`
    Title       string         `json:"title" gorm:"size:128;not null"`
    Description string         `json:"description" gorm:"type:text"`
    Status      string         `json:"status" gorm:"size:20;not null;default:todo;index"`
    CreatedAt   time.Time      `json:"created_at"`
    UpdatedAt   time.Time      `json:"updated_at"`
    DeletedAt   gorm.DeletedAt `json:"-" gorm:"index"`
}
```

---

## 四、数据库连接

`internal/database/mysql.go`：

```go
package database

import (
    "time"

    "gorm.io/driver/mysql"
    "gorm.io/gorm"
)

func NewMySQL(dsn string) (*gorm.DB, error) {
    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
    if err != nil {
        return nil, err
    }

    sqlDB, err := db.DB()
    if err != nil {
        return nil, err
    }

    sqlDB.SetMaxIdleConns(10)
    sqlDB.SetMaxOpenConns(100)
    sqlDB.SetConnMaxLifetime(time.Hour)

    return db, nil
}
```

---

## 五、自动迁移

在 `main.go` 中：

```go
db, err := database.NewMySQL(cfg.Database.DSN)
if err != nil {
    log.Fatal(err)
}

if err := db.AutoMigrate(&model.User{}, &model.Task{}); err != nil {
    log.Fatal(err)
}
```

学习阶段使用 `AutoMigrate` 很方便。生产环境更推荐迁移工具。

---

## 六、为什么任务查询必须带 user_id

错误写法：

```go
db.First(&task, id)
```

这样只要知道任务 ID，就可能查到别人的任务。

正确写法：

```go
db.Where("id = ? AND user_id = ?", id, userID).First(&task)
```

权限边界必须落到数据库查询条件里。

---

## 七、本节练习

完成：

- 创建 `User` 模型。
- 创建 `Task` 模型。
- 初始化数据库连接。
- 执行 `AutoMigrate`。
- 在数据库客户端中确认 users 和 tasks 表已经创建。

---

## 八、本节验收

你应该能够回答：

- `json:"-"` 有什么作用？
- `uniqueIndex` 有什么作用？
- 为什么 tasks 表必须有 `user_id`？
- `DeletedAt` 为什么能实现软删除？
- `AutoMigrate` 适合什么阶段？


