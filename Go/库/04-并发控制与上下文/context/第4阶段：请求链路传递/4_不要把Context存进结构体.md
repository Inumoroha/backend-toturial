# 4. 不要把 Context 存进结构体

本节目标：理解为什么 `context.Context` 通常不应该作为结构体字段保存。

Go 官方和社区都推荐：

```text
不要把 context 存在结构体里。
应该把 context 作为函数参数显式传递。
```

---

## 一、错误示例

```go
type UserService struct {
	ctx  context.Context
	repo *UserRepo
}

func (s *UserService) GetUser(id int64) (*User, error) {
	return s.repo.FindByID(s.ctx, id)
}
```

这个写法的问题很多。

---

## 二、问题一：生命周期混乱

Service 通常是应用级对象。

它可能在程序启动时创建，然后整个服务生命周期内复用。

而 context 通常是请求级对象。

一个请求一个 context。

把请求级对象放进应用级结构体，会导致生命周期混乱。

---

## 三、问题二：可能复用旧请求的 context

假设某次请求创建了 service，并保存了 ctx。

请求结束后 ctx 已经取消。

下一次调用如果继续使用这个 service，就可能拿到一个已经取消的 ctx。

结果是下游数据库查询、HTTP 调用一开始就失败。

---

## 四、问题三：调用关系不清晰

推荐写法：

```go
func (s *UserService) GetUser(ctx context.Context, id int64) (*User, error)
```

调用方一眼能看到：

```text
这个方法支持 context。
调用时必须提供当前请求的 context。
```

如果 context 藏在结构体里，调用关系就不透明。

---

## 五、正确写法

```go
type UserService struct {
	repo *UserRepo
}

func (s *UserService) GetUser(ctx context.Context, id int64) (*User, error) {
	return s.repo.FindByID(ctx, id)
}
```

结构体保存稳定依赖：

- repository。
- logger。
- config。
- client。
- pool。

函数参数传递请求级数据：

- context。
- userID。
- input。
- pagination。

---

## 六、有没有例外

非常少。

某些特殊的长生命周期后台组件可能持有一个根 context，用于组件整体生命周期管理。

但这不适合初学阶段，也不适合普通 service、repository。

你可以先遵守这个规则：

```text
业务分层对象不要保存 context。
方法接收 context。
```

---

## 七、本节达标标准

学完本节后，你应该能够做到：

- 解释为什么 context 不应该存进 service 结构体。
- 区分应用级依赖和请求级数据。
- 使用方法参数传递 ctx。
- 识别旧请求 ctx 被复用的风险。

---

## 八、生命周期对比

可以这样理解：

| 对象 | 生命周期 | 是否适合放结构体 |
| --- | --- | --- |
| `*sql.DB` | 应用级 | 适合 |
| `*http.Client` | 应用级 | 适合 |
| `UserRepo` | 应用级 | 适合 |
| `Logger` | 应用级或请求派生 | 通常适合 |
| `context.Context` | 请求级 | 不适合 |
| `userID` | 业务调用级 | 不适合 |

context 的生命周期通常比 service 短很多。

---

## 九、错误写法的真实后果

假设：

```go
service.ctx = r.Context()
```

第一次请求结束后，`r.Context()` 被取消。

第二次请求复用这个 service，再调用数据库：

```go
db.QueryContext(service.ctx, ...)
```

可能直接返回：

```text
context canceled
```

这个 bug 很隐蔽，因为它和请求时序有关。

---

## 十、正确的依赖注入方式

```go
type UserService struct {
	repo   *UserRepo
	logger *slog.Logger
}

func NewUserService(repo *UserRepo, logger *slog.Logger) *UserService {
	return &UserService{repo: repo, logger: logger}
}

func (s *UserService) GetUser(ctx context.Context, id int64) (*User, error) {
	return s.repo.FindByID(ctx, id)
}
```

结构体保存稳定依赖。

方法参数传递请求级数据。

---

## 十一、例外怎么判断

有些后台组件可能会保存一个 root ctx，例如：

```go
type WorkerGroup struct {
	ctx context.Context
}
```

但这是组件生命周期管理，不是普通业务 service。

初学阶段可以先记住：

```text
handler、service、repository 不保存 context。
```

---

## 十二、本节练习

请重构下面代码：

```go
type OrderService struct {
	ctx context.Context
	repo *OrderRepo
}

func (s *OrderService) Create(orderID int64) error {
	return s.repo.Save(s.ctx, orderID)
}
```

目标：

- 移除结构体里的 ctx。
- `Create` 方法接收 ctx。
- ctx 继续传给 repo。
