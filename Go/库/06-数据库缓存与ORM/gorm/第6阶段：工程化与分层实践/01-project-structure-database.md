# 01 项目结构与数据库初始化

本节目标：围绕「project structure database」建立清晰的 GORM 学习目标，理解它解决的具体后端问题，能够写出可运行示例，并能通过数据库结果或 SQL 日志验证自己的代码是否正确。

单文件 demo 适合入门，但真实项目需要清晰分层。分层不是为了显得复杂，而是为了让数据库代码、业务逻辑和 HTTP 处理各司其职。

## 1. 推荐目录结构

```text
blog-api
├── cmd
│   └── server
│       └── main.go
├── internal
│   ├── config
│   │   └── config.go
│   ├── database
│   │   └── database.go
│   ├── model
│   │   ├── user.go
│   │   ├── article.go
│   │   └── comment.go
│   ├── repository
│   ├── service
│   └── handler
├── go.mod
└── go.sum
```

含义：

- `cmd/server`：程序入口。
- `internal/config`：配置读取。
- `internal/database`：数据库连接初始化。
- `internal/model`：GORM 模型。
- `internal/repository`：数据库访问。
- `internal/service`：业务逻辑。
- `internal/handler`：HTTP 请求处理。

## 2. 配置结构

```go
package config

type Config struct {
    DB DBConfig
}

type DBConfig struct {
    DSN             string
    MaxOpenConns    int
    MaxIdleConns    int
    ConnMaxLifetime int
}
```

学习阶段可以先从环境变量读取：

```go
func Load() Config {
    return Config{
        DB: DBConfig{
            DSN:             os.Getenv("DB_DSN"),
            MaxOpenConns:    100,
            MaxIdleConns:    10,
            ConnMaxLifetime: 3600,
        },
    }
}
```

## 3. 数据库初始化

```go
package database

import (
    "time"

    "gorm.io/driver/mysql"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
)

func New(dsn string) (*gorm.DB, error) {
    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Info),
    })
    if err != nil {
        return nil, err
    }

    sqlDB, err := db.DB()
    if err != nil {
        return nil, err
    }

    sqlDB.SetMaxOpenConns(100)
    sqlDB.SetMaxIdleConns(10)
    sqlDB.SetConnMaxLifetime(time.Hour)

    return db, nil
}
```

## 4. 在 main 中组装

```go
package main

func main() {
    cfg := config.Load()

    db, err := database.New(cfg.DB.DSN)
    if err != nil {
        log.Fatal(err)
    }

    if err := db.AutoMigrate(
        &model.User{},
        &model.Article{},
        &model.Category{},
        &model.Tag{},
        &model.Comment{},
    ); err != nil {
        log.Fatal(err)
    }

    // 初始化 repository、service、handler、router
}
```

真实生产环境里，迁移不一定放在应用启动时自动执行。学习阶段可以保留，方便验证模型。

## 5. 为什么不要到处 `gorm.Open`

错误做法：

```go
func CreateUser() {
    db, _ := gorm.Open(...)
    db.Create(&user)
}
```

问题：

- 每个函数都创建连接，浪费资源。
- 连接池不可控。
- 测试困难。
- 事务不好传递。

正确做法：应用启动时创建一次 `*gorm.DB`，通过构造函数传给需要的组件。

## 本节练习

- [ ] 创建推荐目录结构。
- [ ] 把模型移入 `internal/model`。
- [ ] 写 `database.New`。
- [ ] 在 `main.go` 初始化数据库。
- [ ] 配置连接池参数。
- [ ] 确认程序启动时能连接数据库。
---

## 本节达标标准

学完本节后，你应该能够做到：

- 说清本节主题解决的具体后端开发问题。
- 写出本节涉及的核心 GORM 代码或 SQL 示例。
- 通过 SQL 日志、数据库客户端或测试验证结果是否正确。
- 识别本节列出的常见错误，并知道如何排查。
- 把本节知识放回博客系统或订单系统的真实场景中使用。
