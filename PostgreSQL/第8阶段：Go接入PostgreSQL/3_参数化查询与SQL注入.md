# 3. 参数化查询与 SQL 注入

本节目标：理解 SQL 注入风险，掌握 pgx 中的参数化查询写法，并知道哪些内容不能通过参数占位符替代。

数据库查询里最重要的安全原则之一：

```text
用户输入不能直接拼进 SQL 字符串。
```

---

## 一、什么是 SQL 注入

错误写法：

```go
email := r.URL.Query().Get("email")

sql := "SELECT id, email, username FROM app.users WHERE email = '" + email + "'"
row := pool.QueryRow(ctx, sql)
```

如果用户传入：

```text
' OR '1' = '1
```

拼出来的 SQL 可能变成：

```sql
SELECT id, email, username
FROM app.users
WHERE email = '' OR '1' = '1'
```

这就改变了 SQL 语义。

更危险的场景下，攻击者可能构造删除数据、绕过登录、读取敏感数据的输入。

---

## 二、正确写法：参数化查询

pgx 使用 PostgreSQL 风格占位符：

```text
$1
$2
$3
```

示例：

```go
row := pool.QueryRow(ctx, `
	SELECT id, email, username
	FROM app.users
	WHERE email = $1
`, email)
```

这里 SQL 和参数是分开的：

```text
SQL 模板：WHERE email = $1
参数值：email
```

数据库会把参数当作值处理，而不是当作 SQL 语法处理。

---

## 三、多个参数

```go
row := pool.QueryRow(ctx, `
	SELECT id, email, username
	FROM app.users
	WHERE email = $1
	  AND deleted_at IS NULL
	  AND username = $2
`, email, username)
```

对应关系：

```text
$1 -> email
$2 -> username
```

参数顺序必须匹配。

---

## 四、INSERT 中使用参数

```go
err := pool.QueryRow(ctx, `
	INSERT INTO app.users (email, username, display_name, bio)
	VALUES ($1, $2, $3, $4)
	RETURNING id
`, email, username, displayName, bio).Scan(&id)
```

不要写：

```go
sql := fmt.Sprintf(
	"INSERT INTO app.users (email, username) VALUES ('%s', '%s')",
	email,
	username,
)
```

`fmt.Sprintf` 拼 SQL 是 SQL 注入高发区。

---

## 五、UPDATE 中使用参数

```go
tag, err := pool.Exec(ctx, `
	UPDATE app.users
	SET
		display_name = $2,
		bio = $3,
		updated_at = now()
	WHERE id = $1
	  AND deleted_at IS NULL
`, id, displayName, bio)
if err != nil {
	return err
}

if tag.RowsAffected() == 0 {
	return ErrUserNotFound
}
```

---

## 六、LIKE 查询也要参数化

错误写法：

```go
sql := "SELECT id FROM app.users WHERE username LIKE '%" + keyword + "%'"
```

正确写法：

```go
pattern := "%" + keyword + "%"

rows, err := pool.Query(ctx, `
	SELECT id, email, username
	FROM app.users
	WHERE username ILIKE $1
	ORDER BY id
	LIMIT 20
`, pattern)
```

注意：拼接的是参数值 `pattern`，不是拼接 SQL 语句。

---

## 七、IN 查询怎么写

如果要按一组 ID 查询，可以使用 PostgreSQL 数组和 `ANY`：

```go
ids := []int64{1, 2, 3}

rows, err := pool.Query(ctx, `
	SELECT id, email, username
	FROM app.users
	WHERE id = ANY($1)
`, ids)
```

不要手动拼：

```go
WHERE id IN (1,2,3)
```

更不要把用户输入拼成 `IN (...)` 字符串。

---

## 八、不能参数化的内容

参数只能替代“值”，不能替代表名、列名、排序方向等 SQL 结构。

不能这样：

```go
pool.Query(ctx, `
	SELECT id, email
	FROM $1
`, tableName)
```

也不能这样：

```go
pool.Query(ctx, `
	SELECT id, email
	FROM app.users
	ORDER BY $1
`, orderBy)
```

如果排序字段来自用户输入，应该使用白名单。

示例：

```go
func userOrderBy(input string) string {
	switch input {
	case "created_at":
		return "created_at"
	case "email":
		return "email"
	default:
		return "id"
	}
}
```

然后拼接白名单结果：

```go
orderBy := userOrderBy(input)

sql := `
	SELECT id, email, username
	FROM app.users
	WHERE deleted_at IS NULL
	ORDER BY ` + orderBy + `
	LIMIT $1
`

rows, err := pool.Query(ctx, sql, limit)
```

这里拼接的是后端白名单生成的安全 SQL 片段，不是用户原始输入。

---

## 九、动态排序方向

排序方向也用白名单：

```go
func sortDirection(input string) string {
	switch input {
	case "asc":
		return "ASC"
	case "desc":
		return "DESC"
	default:
		return "DESC"
	}
}
```

使用：

```go
sql := `
	SELECT id, email, username
	FROM app.users
	WHERE deleted_at IS NULL
	ORDER BY created_at ` + sortDirection(inputDirection) + `
	LIMIT $1
`
```

不要直接拼用户输入。

---

## 十、参数化查询的额外好处

除了安全，参数化查询还有其他好处：

- SQL 结构稳定。
- 类型转换更明确。
- 驱动和数据库更容易处理参数。
- 代码可读性更好。

---

## 十一、常见误区

### 1. 只要用户输入不是登录表单，就不用防注入？

不是。

任何外部输入都可能被构造，包括查询参数、请求体、Header、路径参数。

### 2. 数字参数可以直接拼接吗？

不建议。

即使看起来是数字，也应该解析成 Go 数字类型后作为参数传入。

### 3. LIKE 只能拼字符串吗？

不是。

可以拼参数值，例如 `"%" + keyword + "%"`，再作为 `$1` 传入。

### 4. 表名和列名也能用 `$1` 吗？

不能。

参数只能表示值。表名、列名、排序方向要用白名单生成。

---

## 十二、本节达标标准

学完本节后，你应该能够做到：

- 解释 SQL 注入是什么。
- 使用 `$1`、`$2` 编写参数化查询。
- 在 SELECT、INSERT、UPDATE 中传递参数。
- 使用参数化 LIKE 查询。
- 使用 `ANY($1)` 处理一组 ID。
- 知道表名、列名、排序方向不能直接参数化。
- 使用白名单处理动态排序字段。
