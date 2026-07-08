# 2. Handler、Service、Repository 传递 Context

本节目标：掌握 Go 后端分层代码中的 context 传递方式。

推荐分层：

```text
Handler：处理 HTTP 输入输出。
Service：处理业务逻辑。
Repository：处理数据访问。
```

context 应该从 handler 一路传到 repository。

---

## 一、模型定义

```go
type User struct {
	ID   int64
	Name string
}
```

---

## 二、Repository

```go
type UserRepo struct {
	db *sql.DB
}

func (r *UserRepo) FindByID(ctx context.Context, id int64) (*User, error) {
	ctx, cancel := context.WithTimeout(ctx, 300*time.Millisecond)
	defer cancel()

	row := r.db.QueryRowContext(ctx, `
		SELECT id, name
		FROM users
		WHERE id = ?
	`, id)

	var user User
	if err := row.Scan(&user.ID, &user.Name); err != nil {
		return nil, err
	}

	return &user, nil
}
```

Repository 接收 context，并把它传给数据库。

---

## 三、Service

```go
type UserService struct {
	repo *UserRepo
}

func (s *UserService) GetUser(ctx context.Context, id int64) (*User, error) {
	if id <= 0 {
		return nil, fmt.Errorf("invalid user id")
	}

	return s.repo.FindByID(ctx, id)
}
```

Service 不创建根 context。

它只接收上游 context，并传给下游。

---

## 四、Handler

```go
type UserHandler struct {
	service *UserService
}

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	user, err := h.service.GetUser(ctx, 1001)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	_ = json.NewEncoder(w).Encode(user)
}
```

Handler 是 HTTP 世界和业务世界的边界。

它从 `r.Context()` 获取 context，然后传给 service。

---

## 五、为什么 ctx 是第一个参数

Go 社区约定：

```go
func Do(ctx context.Context, arg Arg) error
```

原因：

- 一眼看出函数支持取消和超时。
- 调用链统一。
- 和标准库、常见第三方库风格一致。

不推荐：

```go
func Do(arg Arg, ctx context.Context) error
```

---

## 六、本节练习

请写一个三层结构：

1. `UserHandler.GetUser`
2. `UserService.GetUser`
3. `UserRepo.FindByID`

要求：

- 三层都不要创建 `context.Background()`。
- `ctx` 从 handler 开始向下传递。
- repository 中使用 300ms 子超时。

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 设计带 context 的 service 和 repository 方法。
- 把 `ctx` 作为第一个参数。
- 在 repository 中继续使用上游 context。
- 避免 service 依赖 HTTP 类型。

---

## 八、完整文件结构示例

```text
internal/user/
  model.go
  handler.go
  service.go
  repository.go
```

`model.go`：

```go
type User struct {
	ID   int64  `json:"id"`
	Name string `json:"name"`
}
```

`service.go` 只处理业务规则。

`repository.go` 只处理数据访问。

这样 context 的传递路径非常清楚。

---

## 九、模拟 repository 版本

如果还没有数据库，可以先这样练：

```go
func (r *UserRepo) FindByID(ctx context.Context, id int64) (*User, error) {
	ctx, cancel := context.WithTimeout(ctx, 300*time.Millisecond)
	defer cancel()

	select {
	case <-time.After(100 * time.Millisecond):
		return &User{ID: id, Name: "Tom"}, nil
	case <-ctx.Done():
		return nil, fmt.Errorf("find user stopped: %w", ctx.Err())
	}
}
```

后面接数据库时，只需要把 `select` 换成 `QueryContext`。

---

## 十、链路中的 timeout 放哪里

可以分两层：

```text
handler 或 service：接口总 timeout。
repository：数据库子 timeout。
```

例如：

```go
ctx, cancel := context.WithTimeout(r.Context(), time.Second)
defer cancel()

user, err := h.service.GetUser(ctx, id)
```

repository 再设置：

```go
ctx, cancel := context.WithTimeout(ctx, 300*time.Millisecond)
defer cancel()
```

子 timeout 不会突破父 timeout。

---

## 十一、常见错误

### 1. service 不接收 ctx

```go
func (s *UserService) GetUser(id int64)
```

这样链路断了。

### 2. repo 内部 Background

数据库查询不再受请求取消控制。

### 3. handler 直接访问 db

小 demo 可以，但真实项目不利于维护和测试。

---

## 十二、本节练习

请把示例改成：

1. handler 设置 1 秒总 timeout。
2. repository 设置 300ms 子 timeout。
3. repository 模拟耗时 500ms。
4. handler 判断 `DeadlineExceeded` 返回 504。
5. 把 repository 耗时改成 100ms，确认成功。
