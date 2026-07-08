# 3. Model、Service、Store 设计

本节目标：定义 Todo 项目的核心业务层，让 Handler 不直接操作存储细节。

---

## 一、Model

```go
package todo

type Todo struct {
	ID    int    `json:"id"`
	Title string `json:"title"`
	Done  bool   `json:"done"`
}
```

---

## 二、Store 接口

```go
type Store interface {
	List(ctx context.Context) ([]Todo, error)
	Create(ctx context.Context, title string) (Todo, error)
	Get(ctx context.Context, id int) (Todo, error)
	Update(ctx context.Context, id int, input UpdateInput) (Todo, error)
	Delete(ctx context.Context, id int) error
}
```

这样 Handler 和 Service 不关心数据存在内存还是数据库。

---

## 三、Service

```go
type Service struct {
	store Store
}

func NewService(store Store) *Service {
	return &Service{store: store}
}
```

Service 负责业务规则，例如：

- title 不能为空。
- id 必须合法。
- 更新时处理可选字段。

---

## 四、错误定义

```go
var ErrNotFound = errors.New("todo not found")
var ErrInvalidTitle = errors.New("invalid title")
```

Handler 根据错误映射状态码：

```go
switch {
case errors.Is(err, todo.ErrNotFound):
	writeError(w, 404, "not_found", "todo not found")
case errors.Is(err, todo.ErrInvalidTitle):
	writeError(w, 400, "invalid_request", "title is required")
default:
	writeError(w, 500, "internal_error", "internal server error")
}
```

---

## 五、本节检查点

请确认你能回答：

- Store 接口有什么好处？
- Service 应该放哪些业务规则？
- Handler 为什么不直接操作 map？
- 业务错误如何映射到 HTTP 状态码？

