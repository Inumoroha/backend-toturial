# 4. pgxpool 连接池初始化

本节目标：在 Go 项目中正确初始化 `pgxpool`，并理解连接池为什么重要。

PostgreSQL 是服务端数据库。

Go 服务每次访问数据库都需要连接，如果每个请求都新建连接，会产生明显成本：

```text
建立 TCP 连接。
认证。
初始化会话。
执行查询。
关闭连接。
```

连接池的作用是复用连接。

---

## 一、为什么需要连接池

没有连接池时，请求流程可能是：

```text
HTTP 请求进来
-> 创建数据库连接
-> 执行 SQL
-> 关闭连接
```

有连接池时：

```text
服务启动时创建连接池
-> HTTP 请求进来
-> 从池里借连接
-> 执行 SQL
-> 连接还回池里
```

`pgxpool` 会帮你管理：

- 最大连接数。
- 最小空闲连接。
- 连接健康检查。
- 连接生命周期。
- 并发获取连接。

---

## 二、安装 pgx

```powershell
go get github.com/jackc/pgx/v5
```

常用包：

```go
import (
    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgconn"
    "github.com/jackc/pgx/v5/pgxpool"
)
```

分别用于：

- `pgx`：查询、事务、`ErrNoRows` 等。
- `pgconn`：处理 PostgreSQL 错误。
- `pgxpool`：连接池。

---

## 三、创建 pool.go

创建：

```text
internal/db/pool.go
```

代码：

```go
package db

import (
    "context"
    "time"

    "github.com/jackc/pgx/v5/pgxpool"
)

func NewPool(ctx context.Context, databaseURL string) (*pgxpool.Pool, error) {
    cfg, err := pgxpool.ParseConfig(databaseURL)
    if err != nil {
        return nil, err
    }

    cfg.MaxConns = 10
    cfg.MinConns = 1
    cfg.MaxConnLifetime = time.Hour
    cfg.MaxConnIdleTime = 30 * time.Minute
    cfg.HealthCheckPeriod = time.Minute

    pool, err := pgxpool.NewWithConfig(ctx, cfg)
    if err != nil {
        return nil, err
    }

    pingCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    if err := pool.Ping(pingCtx); err != nil {
        pool.Close()
        return nil, err
    }

    return pool, nil
}
```

这里做了两件重要的事：

```text
创建连接池。
启动时 Ping 数据库，尽早发现连接配置错误。
```

---

## 四、连接池参数解释

### 1. `MaxConns`

最大连接数。

```go
cfg.MaxConns = 10
```

这不是越大越好。

如果服务开了太多连接，PostgreSQL 会承受更多并发压力。

学习项目里可以先用 `10`。

真实项目里要结合：

- PostgreSQL `max_connections`。
- 服务实例数量。
- 查询耗时。
- 峰值并发。

例如有 5 个 Go 服务实例，每个实例 `MaxConns = 20`，总连接上限就是：

```text
5 * 20 = 100
```

### 2. `MinConns`

池里保持的最小连接数。

```go
cfg.MinConns = 1
```

学习项目保持一个即可。

### 3. `MaxConnLifetime`

单个连接最长存活时间。

设置它可以避免连接长期不释放。

### 4. `MaxConnIdleTime`

连接空闲太久后可以关闭。

### 5. `HealthCheckPeriod`

健康检查间隔。

---

## 五、context 的作用

数据库调用都应该带 `context.Context`。

例如：

```go
ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
defer cancel()

row := pool.QueryRow(ctx, `
    SELECT original_url
    FROM app.short_links
    WHERE code = $1
`, code)
```

这样做的好处是：

- HTTP 请求取消时，数据库查询也能取消。
- 查询太慢时可以超时。
- 避免请求堆积。

不要在业务代码里长期使用：

```go
context.Background()
```

HTTP 请求处理里，优先从：

```go
r.Context()
```

继续派生。

---

## 六、连接池放在哪里

连接池应该在程序启动时创建一次，然后注入给 repository：

```go
pool, err := db.NewPool(ctx, cfg.DatabaseURL)
if err != nil {
    log.Fatalf("connect database: %v", err)
}
defer pool.Close()

repo := link.NewRepository(pool)
```

不要每个请求都创建连接池。

错误示例：

```go
func handler(w http.ResponseWriter, r *http.Request) {
    pool, _ := pgxpool.New(r.Context(), databaseURL)
    defer pool.Close()
}
```

这会让连接池失去意义。

---

## 七、repository 持有连接池

创建：

```text
internal/link/repository.go
```

先写结构：

```go
package link

import "github.com/jackc/pgx/v5/pgxpool"

type Repository struct {
    pool *pgxpool.Pool
}

func NewRepository(pool *pgxpool.Pool) *Repository {
    return &Repository{pool: pool}
}
```

后续所有 SQL 方法都挂在 `Repository` 上：

```go
func (r *Repository) Create(ctx context.Context, input CreateLinkInput) (Link, error) {
    // INSERT ...
}
```

---

## 八、测试连接池是否正常

可以临时在 `main.go` 中加入：

```go
var now time.Time
err = pool.QueryRow(ctx, `SELECT now()`).Scan(&now)
if err != nil {
    log.Fatalf("query now: %v", err)
}
log.Printf("database time: %s", now.Format(time.RFC3339))
```

如果能正常输出数据库时间，说明：

- 连接字符串正确。
- PostgreSQL 可访问。
- 用户名密码正确。
- 数据库存在。
- Go 服务能执行查询。

测试完成后可以删除这段临时代码。

---

## 九、常见错误

### 1. `connection refused`

常见原因：

- PostgreSQL 没启动。
- 端口不对。
- Docker 端口没有映射。

检查：

```powershell
docker ps
```

### 2. `password authentication failed`

用户名或密码不对。

检查 `DATABASE_URL`：

```text
postgres://用户名:密码@主机:端口/数据库?sslmode=disable
```

### 3. `database does not exist`

数据库还没创建。

可以用 `psql` 创建：

```sql
CREATE DATABASE learn_pg;
```

### 4. 忘记 `sslmode=disable`

本地 Docker 开发时通常需要：

```text
?sslmode=disable
```

否则可能遇到 SSL 相关错误。

---

## 十、本节达标标准

学完本节后，你应该能够做到：

- 解释连接池的作用。
- 使用 `pgxpool.ParseConfig` 配置连接池。
- 启动时 `Ping` 数据库。
- 在程序退出时 `pool.Close()`。
- 知道不要在每个请求里创建连接池。
- 能把 `*pgxpool.Pool` 注入到 repository。
- 能根据常见错误排查连接问题。

