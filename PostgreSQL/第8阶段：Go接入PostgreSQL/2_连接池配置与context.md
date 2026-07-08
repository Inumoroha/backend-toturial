# 2. 连接池配置与 context

本节目标：理解 `pgxpool` 连接池配置，掌握 `context.Context` 在数据库操作中的作用，并知道如何避免连接泄漏和请求无限等待。

连接池和 `context` 是 Go 后端稳定访问数据库的基础。

---

## 一、为什么需要连接池

数据库连接不是普通变量。

创建一个连接需要：

- 建立网络连接。
- 认证。
- 初始化会话状态。
- 占用数据库服务端资源。

如果每个请求都新建连接：

```text
请求 1 -> 新建连接
请求 2 -> 新建连接
请求 3 -> 新建连接
...
```

高并发时很容易出现：

- 数据库连接数耗尽。
- 请求变慢。
- 数据库压力增大。
- 服务不稳定。

连接池的做法是：

```text
启动时创建池。
请求执行 SQL 时借连接。
执行完归还连接。
```

---

## 二、使用默认配置

最简单写法：

```go
pool, err := pgxpool.New(ctx, databaseURL)
if err != nil {
	return err
}
defer pool.Close()
```

这会使用 pgxpool 默认配置。

学习阶段可以先用默认配置。进入真实项目后，建议明确配置连接数和连接生命周期。

---

## 三、使用 ParseConfig 配置连接池

```go
package db

import (
	"context"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
)

func NewPool(ctx context.Context, databaseURL string) (*pgxpool.Pool, error) {
	config, err := pgxpool.ParseConfig(databaseURL)
	if err != nil {
		return nil, err
	}

	config.MaxConns = 10
	config.MinConns = 2
	config.MaxConnLifetime = time.Hour
	config.MaxConnIdleTime = 30 * time.Minute
	config.HealthCheckPeriod = time.Minute

	pool, err := pgxpool.NewWithConfig(ctx, config)
	if err != nil {
		return nil, err
	}

	if err := pool.Ping(ctx); err != nil {
		pool.Close()
		return nil, err
	}

	return pool, nil
}
```

这段代码做了几件事：

- 解析连接字符串。
- 设置最大连接数。
- 设置最小连接数。
- 设置连接生命周期。
- 创建连接池。
- 启动时 Ping 数据库。

---

## 四、MaxConns 怎么设置

```go
config.MaxConns = 10
```

`MaxConns` 表示连接池最多同时持有多少个连接。

不是越大越好。

设置太小：

```text
请求排队等待连接。
接口变慢。
```

设置太大：

```text
数据库连接数被打满。
数据库上下文切换和内存压力增加。
```

初学阶段可以用：

```text
5 到 20
```

真实项目要结合：

- 服务实例数量。
- PostgreSQL 最大连接数。
- 每个请求使用数据库的时间。
- 是否有 pgbouncer 等连接池代理。

例如：

```text
数据库允许 100 个连接。
有 5 个 Go 服务实例。
每个实例 MaxConns 不应该随便设成 100。
```

---

## 五、context 的作用

数据库方法通常都接收 `context.Context`：

```go
pool.QueryRow(ctx, sql, args...)
pool.Query(ctx, sql, args...)
pool.Exec(ctx, sql, args...)
```

`context` 可以控制：

- 请求取消。
- 超时。
- 链路传递。

Web 请求中通常使用请求自带的 context：

```go
func handler(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	// 用 ctx 查询数据库
}
```

如果客户端断开连接，`r.Context()` 会被取消，数据库操作也有机会停止。

---

## 六、给数据库操作加超时

可以使用：

```go
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()

err := pool.QueryRow(ctx, `
	SELECT now()::text
`).Scan(&now)
```

在 HTTP handler 中：

```go
func handler(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
	defer cancel()

	// 用 ctx 执行数据库操作
}
```

超时不是性能优化本身，而是保护措施。

没有超时的请求可能因为锁等待、慢查询、网络问题一直挂住。

---

## 七、rows 必须关闭

查询多行：

```go
rows, err := pool.Query(ctx, `
	SELECT id, email, username
	FROM app.users
	ORDER BY id
`)
if err != nil {
	return err
}
defer rows.Close()
```

如果不关闭 `rows`，连接可能无法及时归还连接池。

循环结束后还要检查：

```go
if err := rows.Err(); err != nil {
	return err
}
```

完整结构：

```go
rows, err := pool.Query(ctx, sql)
if err != nil {
	return nil, err
}
defer rows.Close()

var users []User
for rows.Next() {
	var u User
	if err := rows.Scan(&u.ID, &u.Email, &u.Username); err != nil {
		return nil, err
	}
	users = append(users, u)
}

if err := rows.Err(); err != nil {
	return nil, err
}

return users, nil
```

---

## 八、什么时候需要手动 Acquire

大多数 CRUD 不需要手动获取连接。

直接用：

```go
pool.QueryRow(ctx, ...)
pool.Query(ctx, ...)
pool.Exec(ctx, ...)
```

就可以。

需要手动 `Acquire` 的场景包括：

- 需要在同一个连接上执行特殊会话级操作。
- 需要访问底层连接。
- 需要做连接级别的准备工作。

示例：

```go
conn, err := pool.Acquire(ctx)
if err != nil {
	return err
}
defer conn.Release()

err = conn.QueryRow(ctx, "SELECT now()::text").Scan(&now)
```

学习阶段不需要频繁使用 `Acquire`。

---

## 九、连接池状态

可以查看连接池状态：

```go
stat := pool.Stat()
fmt.Println("total:", stat.TotalConns())
fmt.Println("acquired:", stat.AcquiredConns())
fmt.Println("idle:", stat.IdleConns())
```

这些指标在排查连接池耗尽时很有用。

如果 `AcquiredConns` 长时间接近 `MaxConns`，可能说明：

- SQL 太慢。
- 事务太长。
- `rows` 没有关闭。
- 请求量确实超过池容量。

---

## 十、常见误区

### 1. MaxConns 越大越好？

不是。

连接越多，数据库压力越大。要结合服务实例数量和数据库能力设置。

### 2. 每个 Repository 都应该创建自己的 pool？

不是。

通常一个应用进程创建一个 pool，然后注入到不同 Repository。

### 3. context 超时能替代 SQL 优化吗？

不能。

超时只能避免无限等待，慢查询仍然要通过索引、SQL、事务设计优化。

### 4. Query 多行不关闭 rows 影响大吗？

影响很大。

连接可能不能及时回到连接池，最终导致连接池耗尽。

---

## 十一、本节达标标准

学完本节后，你应该能够做到：

- 使用 `pgxpool.ParseConfig` 配置连接池。
- 理解 `MaxConns` 不是越大越好。
- 使用 `context.WithTimeout` 控制数据库操作超时。
- 在 HTTP 请求中使用 `r.Context()`。
- 查询多行时正确 `defer rows.Close()`。
- 使用 `rows.Err()` 检查迭代错误。
- 知道大多数场景不需要手动 `Acquire`。
- 使用 `pool.Stat()` 初步观察连接池状态。
