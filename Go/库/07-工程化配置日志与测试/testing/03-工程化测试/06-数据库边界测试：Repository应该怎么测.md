# 6. 数据库边界测试：Repository 应该怎么测

本节目标：学完后你能理解 repository 测试的边界，并用参数化查询保护 SQL 安全。

简短引入：数据库测试是后端项目里最容易写重的部分。service 测业务规则，repository 测 SQL 和数据映射。这里的关键不是“所有 SQL 都测一遍”，而是把会影响数据正确性的边界测清楚。

## 一、为什么需要它

repository 层常见 bug 有：

- SQL 参数顺序写错。
- `Scan` 字段顺序和查询字段不一致。
- 唯一约束错误没有正确处理。
- 事务里某一步失败后没有回滚。
- 把用户输入直接拼进 SQL。

其中最后一个尤其危险。

```text
用户输入不能直接拼进 SQL 字符串。
```

真实项目中通常使用参数化查询，把 SQL 模板和参数分开传给数据库驱动。

## 二、基本用法

这个示例使用 `database/sql` 和 `sqlmock`。`sqlmock` 不连接真实数据库，只验证代码是否按预期执行 SQL，适合 repository 的单元测试。

创建项目：

Windows PowerShell：

```powershell
mkdir repodemo
cd repodemo
go mod init example.com/repodemo
go get github.com/DATA-DOG/go-sqlmock
```

Linux/macOS：

```bash
mkdir repodemo
cd repodemo
go mod init example.com/repodemo
go get github.com/DATA-DOG/go-sqlmock
```

新建 `user_repo.go`：

```go
package repodemo

import (
	"context"
	"database/sql"
)

type User struct {
	ID    int64
	Email string
}

type UserRepo struct {
	db *sql.DB
}

func NewUserRepo(db *sql.DB) *UserRepo {
	return &UserRepo{db: db}
}

func (r *UserRepo) FindByEmail(ctx context.Context, email string) (User, error) {
	const query = `SELECT id, email FROM users WHERE email = ?`

	var user User
	err := r.db.QueryRowContext(ctx, query, email).Scan(&user.ID, &user.Email)
	if err != nil {
		return User{}, err
	}
	return user, nil
}
```

## 三、关键参数/语法/代码结构

`?` 是参数占位符。不同数据库驱动占位符可能不同，例如 PostgreSQL 常用 `$1`、`$2`。但原则一样：用户输入作为参数传入，不拼接进 SQL 字符串。

`QueryRowContext(ctx, query, email)` 里第三个参数才是用户输入。

`Scan(&user.ID, &user.Email)` 的顺序必须和 `SELECT id, email` 一致。

`sql.ErrNoRows` 是查询不到数据时的常见错误。真实项目中通常会把它转换成业务层能理解的错误，例如 `ErrUserNotFound`。

## 四、真实后端场景示例

新建 `user_repo_test.go`：

```go
package repodemo

import (
	"context"
	"database/sql"
	"testing"

	"github.com/DATA-DOG/go-sqlmock"
)

func TestUserRepo_FindByEmail(t *testing.T) {
	db, mock, err := sqlmock.New()
	if err != nil {
		t.Fatalf("create sqlmock: %v", err)
	}
	t.Cleanup(func() {
		_ = db.Close()
	})

	repo := NewUserRepo(db)

	rows := sqlmock.NewRows([]string{"id", "email"}).
		AddRow(int64(1), "alice@example.com")

	mock.ExpectQuery(`SELECT id, email FROM users WHERE email = \?`).
		WithArgs("alice@example.com").
		WillReturnRows(rows)

	got, err := repo.FindByEmail(context.Background(), "alice@example.com")
	if err != nil {
		t.Fatalf("FindByEmail() error: %v", err)
	}
	if got.ID != 1 || got.Email != "alice@example.com" {
		t.Fatalf("user = %+v", got)
	}
	if err := mock.ExpectationsWereMet(); err != nil {
		t.Fatalf("sql expectations: %v", err)
	}
}

func TestUserRepo_FindByEmail_NotFound(t *testing.T) {
	db, mock, err := sqlmock.New()
	if err != nil {
		t.Fatalf("create sqlmock: %v", err)
	}
	t.Cleanup(func() {
		_ = db.Close()
	})

	repo := NewUserRepo(db)

	mock.ExpectQuery(`SELECT id, email FROM users WHERE email = \?`).
		WithArgs("missing@example.com").
		WillReturnError(sql.ErrNoRows)

	_, err = repo.FindByEmail(context.Background(), "missing@example.com")
	if err != sql.ErrNoRows {
		t.Fatalf("error = %v, want sql.ErrNoRows", err)
	}
	if err := mock.ExpectationsWereMet(); err != nil {
		t.Fatalf("sql expectations: %v", err)
	}
}
```

运行：

```powershell
go test ./...
```

Linux/macOS：

```bash
go test ./...
```

这个测试验证了三件事：SQL 被执行、参数没有拼接、查询结果正确映射到了结构体。

## 五、注意点

`sqlmock` 测的是“你的代码是否按预期调用数据库接口”，不是测试真实数据库行为。索引、约束、事务隔离级别、SQL 方言差异，仍然需要少量集成测试验证。

repository 单元测试不要写太多重复用例。优先覆盖：

- 参数化查询。
- 字段映射。
- 查询不到数据。
- 唯一约束或外键错误的转换。
- 事务提交和回滚。

真实项目中，迁移工具也很重要。不要在测试里临时写一堆 `CREATE TABLE` 字符串，然后和生产迁移脱节。集成测试应该尽量复用正式迁移文件。

## 六、常见误区

- 把 SQL 写成 `"... WHERE email = '" + email + "'"`：这是 SQL 注入风险。
- 只测成功查询：查不到数据、重复数据、数据库错误才是后端常见边界。
- 过度依赖 mock：mock 通过不代表真实数据库一定通过。
- 测试里不检查 `ExpectationsWereMet`：SQL 没按预期执行也可能漏掉。
- repository 返回 HTTP 状态码：repository 不应该知道 HTTP，错误转换应放在 service 或 handler。

## 七、本节达标标准

- 能说明 repository 测试和 service 测试的边界。
- 能写参数化查询，避免拼接用户输入。
- 能用 `sqlmock` 测 SQL 调用和参数。
- 能处理 `sql.ErrNoRows`。
- 能说明哪些数据库行为必须用集成测试验证。

