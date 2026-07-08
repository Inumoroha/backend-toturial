# 3. Go 项目结构与配置

本节目标：搭建短链接服务的 Go 项目结构，并准备数据库连接配置。

这一节先不急着写业务 SQL，而是把项目骨架搭好。

---

## 一、初始化项目

创建目录：

```powershell
mkdir shortener
cd shortener
```

初始化 Go module：

```powershell
go mod init example.com/shortener
```

安装 `pgx`：

```powershell
go get github.com/jackc/pgx/v5
```

如果使用 `pgxpool`，包路径仍然在 `pgx/v5` 里：

```go
import "github.com/jackc/pgx/v5/pgxpool"
```

---

## 二、推荐目录结构

建议使用：

```text
shortener/
  go.mod
  go.sum
  cmd/
    api/
      main.go
  internal/
    config/
      config.go
    db/
      pool.go
    link/
      model.go
      repository.go
      service.go
      handler.go
      code.go
      errors.go
  migrations/
    00001_create_short_links.sql
  README.md
  Makefile
```

解释：

- `cmd/api/main.go`：应用入口，负责组装配置、数据库、路由和启动 HTTP 服务。
- `internal/config`：读取环境变量。
- `internal/db`：创建和关闭连接池。
- `internal/link`：短链接模块。
- `migrations`：数据库迁移文件。

`internal` 是 Go 的特殊目录。

放在 `internal` 里的包只能被当前 module 内部引用，适合放项目内部业务代码。

---

## 三、配置项设计

第一版需要这些配置：

```text
HTTP_ADDR
DATABASE_URL
BASE_URL
```

含义：

| 配置 | 示例 | 作用 |
| --- | --- | --- |
| `HTTP_ADDR` | `:8080` | HTTP 服务监听地址 |
| `DATABASE_URL` | `postgres://postgres:postgres@localhost:5432/learn_pg?sslmode=disable` | PostgreSQL 连接字符串 |
| `BASE_URL` | `http://localhost:8080` | 拼接返回给用户的短链接 |

---

## 四、config.go

创建：

```text
internal/config/config.go
```

代码：

```go
package config

import "os"

type Config struct {
    HTTPAddr    string
    DatabaseURL string
    BaseURL     string
}

func Load() Config {
    return Config{
        HTTPAddr:    env("HTTP_ADDR", ":8080"),
        DatabaseURL: env("DATABASE_URL", "postgres://postgres:postgres@localhost:5432/learn_pg?sslmode=disable"),
        BaseURL:     env("BASE_URL", "http://localhost:8080"),
    }
}

func env(key string, fallback string) string {
    value := os.Getenv(key)
    if value == "" {
        return fallback
    }
    return value
}
```

这里为了学习方便给了默认值。

生产项目里，通常建议 `DATABASE_URL` 必须显式配置，避免误连数据库。

---

## 五、main.go 的职责

`main.go` 不应该写一大堆业务 SQL。

它主要做：

```text
读取配置。
创建数据库连接池。
创建 repository。
创建 service。
创建 handler。
注册路由。
启动 HTTP 服务。
```

示例：

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "example.com/shortener/internal/config"
    "example.com/shortener/internal/db"
    "example.com/shortener/internal/link"
)

func main() {
    cfg := config.Load()

    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()

    pool, err := db.NewPool(ctx, cfg.DatabaseURL)
    if err != nil {
        log.Fatalf("connect database: %v", err)
    }
    defer pool.Close()

    repo := link.NewRepository(pool)
    service := link.NewService(repo, cfg.BaseURL)
    handler := link.NewHandler(service)

    mux := http.NewServeMux()
    handler.RegisterRoutes(mux)

    server := &http.Server{
        Addr:              cfg.HTTPAddr,
        Handler:           mux,
        ReadHeaderTimeout: 5 * time.Second,
    }

    go func() {
        log.Printf("listening on %s", cfg.HTTPAddr)
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("listen: %v", err)
        }
    }()

    <-ctx.Done()

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := server.Shutdown(shutdownCtx); err != nil {
        log.Printf("shutdown server: %v", err)
    }
}
```

---

## 六、为什么要分 repository、service、handler

可以把所有代码都写在 `handler.go` 里吗？

学习时可以，但项目很快会乱。

推荐分层：

```text
handler    处理 HTTP 请求和响应。
service    处理业务规则。
repository 处理数据库 SQL。
```

对应关系：

| 层 | 关注点 |
| --- | --- |
| handler | JSON、状态码、路由参数、查询参数 |
| service | URL 校验、短码生成、重试策略、业务错误 |
| repository | SQL、事务、扫描结果、数据库错误 |

这样做的好处是：

- SQL 不散落在 HTTP 层。
- 业务规则更容易测试。
- 数据库错误可以在 repository 层先转换。
- handler 不需要知道太多 PostgreSQL 细节。

---

## 七、模型文件 model.go

创建：

```text
internal/link/model.go
```

代码：

```go
package link

import (
    "time"

    "github.com/google/uuid"
)

type Link struct {
    ID          uuid.UUID `json:"id"`
    Code        string    `json:"code"`
    OriginalURL string    `json:"original_url"`
    ShortURL    string    `json:"short_url,omitempty"`
    VisitCount  int64     `json:"visit_count"`
    CreatedAt   time.Time `json:"created_at"`
    UpdatedAt   time.Time `json:"updated_at"`
}
```

这里用了 `github.com/google/uuid`。

安装：

```powershell
go get github.com/google/uuid
```

也可以不用第三方 UUID 包，把 `ID` 暂时写成字符串：

```go
ID string `json:"id"`
```

不过在真实 Go 项目里，用专门的 UUID 类型会更清晰。

---

## 八、错误文件 errors.go

创建：

```text
internal/link/errors.go
```

代码：

```go
package link

import "errors"

var (
    ErrNotFound      = errors.New("link not found")
    ErrInvalidURL    = errors.New("invalid url")
    ErrCodeConflict  = errors.New("short code conflict")
    ErrRetryExceeded = errors.New("short code retry exceeded")
)
```

业务层和 HTTP 层都可以使用这些错误判断。

例如：

```go
if errors.Is(err, link.ErrNotFound) {
    http.NotFound(w, r)
    return
}
```

---

## 九、启动前检查

写业务代码之前，先保证：

- PostgreSQL 已启动。
- 数据库已创建。
- 迁移已执行。
- `DATABASE_URL` 正确。
- `go mod tidy` 能正常完成。

常用命令：

```powershell
$env:DATABASE_URL="postgres://postgres:postgres@localhost:5432/learn_pg?sslmode=disable"
go mod tidy
go run ./cmd/api
```

---

## 十、常见误区

### 1. main.go 写所有逻辑更快？

一开始看起来快，但很快会变成大文件。

项目落地阶段要开始练习分层。

### 2. 配置可以写死在代码里？

学习阶段可以给默认值，但连接字符串、监听地址、基础域名都应该支持环境变量。

### 3. service 层是不是多余？

如果只是 CRUD，看起来多余。

但短链接创建需要 URL 校验、短码生成、冲突重试，这些都不属于 HTTP，也不属于 SQL，所以 service 层很合适。

---

## 十一、本节达标标准

学完本节后，你应该能够做到：

- 初始化 Go module。
- 建立推荐项目目录。
- 读取 `HTTP_ADDR`、`DATABASE_URL`、`BASE_URL`。
- 理解 `handler`、`service`、`repository` 的职责。
- 写出 `main.go` 的组装逻辑。
- 知道业务错误应该集中定义，而不是到处写字符串。

