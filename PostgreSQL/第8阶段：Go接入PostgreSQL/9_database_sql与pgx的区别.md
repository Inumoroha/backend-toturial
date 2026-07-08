# 9. database/sql 与 pgx 的区别

本节目标：理解 Go 标准库 `database/sql` 和 `pgx` 的关系，知道什么时候用通用接口，什么时候用 PostgreSQL 原生能力。

Go 的数据库生态里有两个常见概念：

```text
database/sql
pgx
```

它们不是简单的谁替代谁，而是定位不同。

---

## 一、database/sql 是什么

`database/sql` 是 Go 标准库提供的通用数据库接口。

它定义了一套统一抽象：

```go
*sql.DB
*sql.Tx
*sql.Rows
*sql.Row
```

不同数据库通过驱动接入它：

- PostgreSQL
- MySQL
- SQLite
- SQL Server

使用 `database/sql` 的好处是：

```text
代码依赖的是 Go 标准库抽象，而不是某个数据库驱动的原生 API。
```

---

## 二、pgx 是什么

`pgx` 是 PostgreSQL 专用的 Go 驱动和工具包。

它更贴近 PostgreSQL 原生能力，例如：

- PostgreSQL 类型支持。
- `pgxpool` 连接池。
- `CopyFrom`。
- 批处理。
- Listen/Notify。
- 更直接的 PostgreSQL 错误信息。

如果项目明确使用 PostgreSQL，直接使用 pgx 很自然。

---

## 三、pgx 也可以接入 database/sql

pgx 提供 `stdlib` 包，可以作为 `database/sql` 驱动使用。

也就是说，pgx 有两种用法：

```text
1. 原生 pgx API。
2. 通过 pgx/stdlib 接入 database/sql。
```

本教程重点学习原生 pgx API：

```go
github.com/jackc/pgx/v5/pgxpool
```

因为目标是明确接入 PostgreSQL。

---

## 四、database/sql 示例

大致写法：

```go
package main

import (
	"context"
	"database/sql"
	"log"

	_ "github.com/jackc/pgx/v5/stdlib"
)

func main() {
	db, err := sql.Open("pgx", "postgres://postgres:postgres@localhost:5432/learn_pg?sslmode=disable")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	ctx := context.Background()
	if err := db.PingContext(ctx); err != nil {
		log.Fatal(err)
	}

	var now string
	err = db.QueryRowContext(ctx, "SELECT now()::text").Scan(&now)
	if err != nil {
		log.Fatal(err)
	}
}
```

注意：

```go
_ "github.com/jackc/pgx/v5/stdlib"
```

这是注册 pgx 的 `database/sql` 驱动。

---

## 五、pgxpool 示例

```go
package main

import (
	"context"
	"log"

	"github.com/jackc/pgx/v5/pgxpool"
)

func main() {
	ctx := context.Background()

	pool, err := pgxpool.New(ctx, "postgres://postgres:postgres@localhost:5432/learn_pg?sslmode=disable")
	if err != nil {
		log.Fatal(err)
	}
	defer pool.Close()

	if err := pool.Ping(ctx); err != nil {
		log.Fatal(err)
	}

	var now string
	err = pool.QueryRow(ctx, "SELECT now()::text").Scan(&now)
	if err != nil {
		log.Fatal(err)
	}
}
```

两种写法都能访问 PostgreSQL。

区别在于你使用的是通用标准接口，还是 PostgreSQL 专用接口。

---

## 六、连接池区别

`database/sql` 的 `*sql.DB` 本身就是连接池，不是单个连接。

它通过方法配置：

```go
db.SetMaxOpenConns(10)
db.SetMaxIdleConns(2)
db.SetConnMaxLifetime(time.Hour)
```

`pgxpool` 使用：

```go
config.MaxConns = 10
config.MinConns = 2
config.MaxConnLifetime = time.Hour
```

两者都有连接池能力。

不要误以为 `database/sql` 每次都新建连接。

---

## 七、错误处理区别

使用原生 pgx 时，可以直接处理：

```go
var pgErr *pgconn.PgError
if errors.As(err, &pgErr) {
	// pgErr.Code
	// pgErr.ConstraintName
}
```

使用 `database/sql` + pgx stdlib 时，也可能拿到来自 pgx 的底层错误，但中间多了一层通用抽象。

如果你非常依赖 PostgreSQL 错误细节，原生 pgx 写起来更直接。

---

## 八、PostgreSQL 类型支持

pgx 对 PostgreSQL 类型支持更直接，例如：

- `uuid`
- `jsonb`
- array
- range
- inet
- numeric
- timestamptz

使用 `database/sql` 也能处理很多类型，但有时需要更多转换或第三方类型。

如果项目大量使用 PostgreSQL 特色类型，pgx 原生 API 更顺手。

---

## 九、什么时候用 database/sql

适合：

- 想依赖 Go 标准库接口。
- 项目需要支持多种数据库。
- 团队已有 `database/sql` 经验。
- 使用的库基于 `database/sql`。
- 只需要通用 SQL 能力。

---

## 十、什么时候用 pgx

适合：

- 项目明确只使用 PostgreSQL。
- 想使用 PostgreSQL 原生类型和能力。
- 需要更直接的错误处理。
- 需要 `CopyFrom` 等高性能批量写入。
- 需要 Listen/Notify 等 PostgreSQL 特性。
- 想使用 pgxpool。

新项目如果明确选择 PostgreSQL，可以优先学习 pgx 和 pgxpool。

---

## 十一、能不能混用

技术上可以，但不建议在同一个项目里随意混用两套访问方式。

混用会带来：

- 连接池重复。
- 事务边界混乱。
- 错误处理不统一。
- 代码风格不一致。

建议一个项目确定一种主要访问方式。

如果使用框架或库必须接 `database/sql`，可以考虑使用 pgx 的 stdlib 作为驱动。

---

## 十二、和 ORM 的关系

还有一些工具：

- GORM
- sqlc
- sqlx
- ent

它们和 pgx、`database/sql` 不是同一层的问题。

例如：

- GORM 是 ORM。
- sqlc 是根据 SQL 生成类型安全 Go 代码。
- sqlx 是 `database/sql` 的扩展。
- ent 是实体框架。

底层仍然要通过驱动连接数据库。

学习阶段先掌握 pgx 的基础能力，再学这些工具会更稳。

---

## 十三、常见误区

### 1. database/sql 是驱动吗？

不是。

它是标准库接口。真正连接 PostgreSQL 还需要驱动，例如 pgx stdlib。

### 2. *sql.DB 是单个连接吗？

不是。

`*sql.DB` 是连接池。

### 3. pgx 只能单连接使用吗？

不是。

Web 服务常用 `pgxpool`。

### 4. 使用 pgx 就不能用 ORM 吗？

不绝对。

但 ORM 或代码生成工具各自有自己的驱动适配方式，要看具体工具支持。

---

## 十四、本节达标标准

学完本节后，你应该能够做到：

- 解释 `database/sql` 是 Go 标准库数据库接口。
- 解释 pgx 是 PostgreSQL 专用驱动和工具包。
- 知道 pgx 可以通过 stdlib 接入 `database/sql`。
- 知道 `*sql.DB` 和 `pgxpool.Pool` 都是连接池。
- 判断什么时候适合用 `database/sql`。
- 判断什么时候适合用原生 pgx。
- 知道同一项目中不要随意混用多套数据库访问方式。
