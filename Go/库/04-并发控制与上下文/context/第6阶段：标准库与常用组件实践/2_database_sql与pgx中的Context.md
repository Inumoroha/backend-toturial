# 2. database/sql 与 pgx 中的 Context

本节目标：比较 `database/sql` 和 `pgx` 中 context 的用法。

数据库是 Go 后端中最重要的外部依赖之一。

数据库调用必须接收 context。

---

## 一、database/sql

常用方法：

```go
db.QueryContext(ctx, sql, args...)
db.QueryRowContext(ctx, sql, args...)
db.ExecContext(ctx, sql, args...)
db.BeginTx(ctx, opts)
```

示例：

```go
func createUser(ctx context.Context, db *sql.DB, email string) error {
	ctx, cancel := context.WithTimeout(ctx, 500*time.Millisecond)
	defer cancel()

	_, err := db.ExecContext(ctx, `
		INSERT INTO users(email)
		VALUES (?)
	`, email)
	return err
}
```

---

## 二、pgx / pgxpool

常用方法：

```go
pool.Query(ctx, sql, args...)
pool.QueryRow(ctx, sql, args...)
pool.Exec(ctx, sql, args...)
pool.Begin(ctx)
```

示例：

```go
func createUser(ctx context.Context, pool *pgxpool.Pool, email string) error {
	ctx, cancel := context.WithTimeout(ctx, 500*time.Millisecond)
	defer cancel()

	_, err := pool.Exec(ctx, `
		INSERT INTO app.users(email)
		VALUES ($1)
	`, email)
	return err
}
```

---

## 三、事务中的 context

```go
tx, err := db.BeginTx(ctx, nil)
if err != nil {
	return err
}
defer tx.Rollback()

if _, err := tx.ExecContext(ctx, "..."); err != nil {
	return err
}

return tx.Commit()
```

事务里也要继续使用同一个 ctx。

如果请求取消，事务中的查询应该停止，然后回滚。

---

## 四、不要滥用短超时

数据库超时要结合业务。

不是所有查询都应该写死 100ms。

需要考虑：

- 查询复杂度。
- 是否走索引。
- 是否在事务里。
- 用户接口还是后台任务。
- 上游请求总超时。

超时太短会造成大量误伤。

超时太长又失去保护意义。

---

## 五、本节达标标准

学完本节后，你应该能够做到：

- 使用 `QueryContext`、`ExecContext`。
- 使用 pgx 的 `Query`、`Exec` 传入 ctx。
- 在事务中传递 ctx。
- 设计合理的数据库查询超时。

---

## 六、完整 Repository 示例

下面以 `database/sql` 为例：

```go
type User struct {
	ID    int64
	Email string
	Name  string
}

type UserRepo struct {
	db *sql.DB
}

func (r *UserRepo) FindByID(ctx context.Context, id int64) (*User, error) {
	ctx, cancel := context.WithTimeout(ctx, 300*time.Millisecond)
	defer cancel()

	row := r.db.QueryRowContext(ctx, `
		SELECT id, email, name
		FROM users
		WHERE id = ?
	`, id)

	var user User
	if err := row.Scan(&user.ID, &user.Email, &user.Name); err != nil {
		if ctxErr := ctx.Err(); ctxErr != nil {
			return nil, fmt.Errorf("find user timeout or canceled: %w", ctxErr)
		}
		return nil, err
	}

	return &user, nil
}
```

这里有几个关键点：

- 方法接收上游 ctx。
- 查询前派生 300ms 子 timeout。
- 使用 `QueryRowContext`。
- 失败时检查 `ctx.Err()`，保留取消或超时根因。

---

## 七、多行查询完整结构

```go
func (r *UserRepo) List(ctx context.Context) ([]User, error) {
	ctx, cancel := context.WithTimeout(ctx, 500*time.Millisecond)
	defer cancel()

	rows, err := r.db.QueryContext(ctx, `
		SELECT id, email, name
		FROM users
		ORDER BY id DESC
		LIMIT 20
	`)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var users []User
	for rows.Next() {
		var user User
		if err := rows.Scan(&user.ID, &user.Email, &user.Name); err != nil {
			return nil, err
		}
		users = append(users, user)
	}

	if err := rows.Err(); err != nil {
		if ctxErr := ctx.Err(); ctxErr != nil {
			return nil, fmt.Errorf("list users stopped: %w", ctxErr)
		}
		return nil, err
	}

	return users, nil
}
```

多行查询一定要：

```text
defer rows.Close()
循环后检查 rows.Err()
```

---

## 八、事务完整结构

```go
func (r *UserRepo) UpdateName(ctx context.Context, id int64, name string) error {
	ctx, cancel := context.WithTimeout(ctx, time.Second)
	defer cancel()

	tx, err := r.db.BeginTx(ctx, nil)
	if err != nil {
		return err
	}
	defer tx.Rollback()

	if _, err := tx.ExecContext(ctx, `
		UPDATE users
		SET name = ?
		WHERE id = ?
	`, name, id); err != nil {
		return err
	}

	return tx.Commit()
}
```

真实项目中如果 `Commit` 也需要 context，可以使用具体驱动提供的事务接口，例如 pgx 的 `Commit(ctx)`。

---

## 九、常见错误

### 1. 查询没有超时

慢查询、锁等待可能让请求长期挂住。

### 2. Query 多行忘记 rows.Close

连接不能及时归还连接池，最终可能导致连接池耗尽。

### 3. 事务中某一步失败但没有 Rollback

事务会持有连接和锁，必须确保失败时回滚。

### 4. 在 repo 中创建 Background

这会切断 handler 传来的请求生命周期。

---

## 十、本节练习

请写一个 `OrderRepo.ListByUserID(ctx, userID)`：

1. 接收 ctx 和 userID。
2. 设置 500ms 查询 timeout。
3. 使用多行查询。
4. 正确关闭 rows。
5. 检查 `rows.Err()`。
6. 超时时返回包装后的 `context.DeadlineExceeded`。
