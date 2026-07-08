# 6. Go 中使用事务

本节目标：掌握 pgx 中事务的基本写法，能在 Go 代码中正确提交、回滚，并知道事务边界应该和业务动作对齐。

事务在 Go 代码中的核心原则和 SQL 中一样：

```text
多条数据库操作必须一起成功或一起失败时，使用事务。
```

---

## 一、基本事务写法

```go
func (r *Repository) DoSomething(ctx context.Context) error {
	tx, err := r.pool.Begin(ctx)
	if err != nil {
		return err
	}
	defer tx.Rollback(ctx)

	_, err = tx.Exec(ctx, `...`)
	if err != nil {
		return err
	}

	_, err = tx.Exec(ctx, `...`)
	if err != nil {
		return err
	}

	return tx.Commit(ctx)
}
```

关键点：

- `Begin` 开启事务。
- `defer tx.Rollback(ctx)` 保证出错时回滚。
- 所有事务内 SQL 都使用 `tx`，不要再用 `pool`。
- 最后 `Commit`。

`Commit` 成功后，延迟执行的 `Rollback` 会发现事务已经关闭，通常可以忽略。

---

## 二、事务里不能混用 pool

错误写法：

```go
tx, err := r.pool.Begin(ctx)
if err != nil {
	return err
}
defer tx.Rollback(ctx)

_, err = tx.Exec(ctx, `UPDATE ...`)
if err != nil {
	return err
}

// 错误：这条不在事务里
_, err = r.pool.Exec(ctx, `INSERT ...`)
if err != nil {
	return err
}

return tx.Commit(ctx)
```

事务开启后，属于这个事务的 SQL 必须通过 `tx` 执行。

否则有些语句在事务内，有些语句在事务外，原子性就被破坏了。

---

## 三、示例：注册用户并初始化偏好

准备表：

```sql
CREATE TABLE app.user_preferences (
    user_id bigint PRIMARY KEY,
    theme text NOT NULL DEFAULT 'light',
    language text NOT NULL DEFAULT 'zh-CN',
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT user_preferences_user_id_fk
        FOREIGN KEY (user_id)
        REFERENCES app.users (id)
        ON DELETE CASCADE
);
```

业务动作：

```text
创建用户。
初始化用户偏好。
```

这两步应该放在一个事务里。

Go 代码：

```go
func (r *Repository) Register(ctx context.Context, p CreateUserParams) (*User, error) {
	tx, err := r.pool.Begin(ctx)
	if err != nil {
		return nil, err
	}
	defer tx.Rollback(ctx)

	var u User
	err = tx.QueryRow(ctx, `
		INSERT INTO app.users (email, username, display_name, bio)
		VALUES ($1, $2, $3, $4)
		RETURNING id, email, username, display_name, bio, created_at, updated_at
	`, p.Email, p.Username, p.DisplayName, p.Bio).Scan(
		&u.ID,
		&u.Email,
		&u.Username,
		&u.DisplayName,
		&u.Bio,
		&u.CreatedAt,
		&u.UpdatedAt,
	)
	if err != nil {
		return nil, err
	}

	_, err = tx.Exec(ctx, `
		INSERT INTO app.user_preferences (user_id)
		VALUES ($1)
	`, u.ID)
	if err != nil {
		return nil, err
	}

	if err := tx.Commit(ctx); err != nil {
		return nil, err
	}

	return &u, nil
}
```

如果初始化偏好失败，用户也会回滚，不会留下半成品用户。

---

## 四、事务隔离级别

可以指定事务选项：

```go
tx, err := r.pool.BeginTx(ctx, pgx.TxOptions{
	IsoLevel: pgx.ReadCommitted,
})
```

常见隔离级别：

```go
pgx.ReadCommitted
pgx.RepeatableRead
pgx.Serializable
```

大多数业务使用 PostgreSQL 默认的 `READ COMMITTED` 就够了。

需要稳定快照或更强一致性时，再考虑更高隔离级别，并准备处理重试。

---

## 五、事务辅助函数

为了减少重复代码，可以封装：

```go
func (r *Repository) withTx(ctx context.Context, fn func(pgx.Tx) error) error {
	tx, err := r.pool.Begin(ctx)
	if err != nil {
		return err
	}
	defer tx.Rollback(ctx)

	if err := fn(tx); err != nil {
		return err
	}

	return tx.Commit(ctx)
}
```

使用：

```go
err := r.withTx(ctx, func(tx pgx.Tx) error {
	_, err := tx.Exec(ctx, `UPDATE ...`)
	if err != nil {
		return err
	}

	_, err = tx.Exec(ctx, `INSERT ...`)
	if err != nil {
		return err
	}

	return nil
})
```

这种模式可以让事务代码更统一。

---

## 六、事务中处理错误

一旦事务中的 SQL 返回错误，通常应该返回错误，让 `defer Rollback` 回滚。

不要在同一个事务中无视错误继续执行。

PostgreSQL 中，如果事务里某条语句失败，事务可能进入失败状态。继续执行其他 SQL 通常只会得到更多错误。

正确思路：

```text
遇到错误。
返回错误。
回滚事务。
由上层决定是否重试或返回给用户。
```

---

## 七、哪些事务错误可以重试

常见可重试错误：

- 死锁。
- 序列化失败。
- 某些锁超时。

如果遇到序列化失败，应该：

```text
回滚当前事务。
重新开启事务。
重新执行完整业务逻辑。
```

不要只重试最后一条 SQL。

---

## 八、事务里不要做什么

不要在事务中：

- 调用外部 HTTP API。
- 等待用户输入。
- 发送短信邮件。
- 做耗时计算。
- 执行不必要的慢查询。

事务越长，持锁越久，占用连接越久，越容易造成连接池耗尽和锁等待。

---

## 九、context 超时和事务

事务也应该使用带超时的 context。

```go
ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
defer cancel()

tx, err := r.pool.Begin(ctx)
```

如果 context 超时，事务中的数据库操作会被取消。

仍然要确保回滚逻辑执行。

---

## 十、常见误区

### 1. defer Rollback 会不会把 Commit 后的数据回滚？

不会。

`Commit` 成功后事务已经结束，后续 `Rollback` 会失败但不会撤销已提交数据。

### 2. 事务里可以混用 tx 和 pool 吗？

不可以。

属于同一个事务的 SQL 必须都通过 `tx` 执行。

### 3. 事务越大越安全吗？

不是。

事务越大，锁和连接占用越久。事务应该只包住必须原子完成的数据库操作。

### 4. 事务错误后继续执行 SQL 可以吗？

不建议。

通常应该回滚并返回错误。

---

## 十一、本节达标标准

学完本节后，你应该能够做到：

- 使用 `pool.Begin` 开启事务。
- 使用 `tx.Commit` 提交事务。
- 使用 `defer tx.Rollback` 兜底回滚。
- 知道事务内 SQL 必须使用 `tx`。
- 使用 `BeginTx` 设置隔离级别。
- 封装简单事务辅助函数。
- 知道事务中不要做外部调用和慢操作。
- 知道可重试事务错误要重试完整事务。
