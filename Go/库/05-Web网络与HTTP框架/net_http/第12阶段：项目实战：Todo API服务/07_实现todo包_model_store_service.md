# 7. 实现 todo 包：model、store、service

本节目标：实现 Todo 项目的业务核心。先不碰 HTTP，先把 Todo 的数据结构、内存存储和业务规则写清楚。

---

## 一、model.go

文件：`internal/todo/model.go`

```go
package todo

type Todo struct {
	ID    int    `json:"id"`
	Title string `json:"title"`
	Done  bool   `json:"done"`
}

type UpdateInput struct {
	Title *string
	Done  *bool
}
```

`UpdateInput` 使用指针，是为了区分：

```text
字段没传
字段传了零值
```

例如 `done=false` 是一个有效更新，不能被当成“没传”。

---

## 二、errors.go

文件：`internal/todo/errors.go`

```go
package todo

import "errors"

var ErrNotFound = errors.New("todo not found")
var ErrInvalidTitle = errors.New("invalid title")
```

业务层返回业务错误，HTTP 层再把它映射成状态码。

---

## 三、store.go

文件：`internal/todo/store.go`

```go
package todo

import "context"

type Store interface {
	List(ctx context.Context) ([]Todo, error)
	Create(ctx context.Context, title string) (Todo, error)
	Get(ctx context.Context, id int) (Todo, error)
	Update(ctx context.Context, id int, input UpdateInput) (Todo, error)
	Delete(ctx context.Context, id int) error
}
```

即使内存 Store 暂时用不到 context，也保留它。以后换数据库时，`QueryContext`、`ExecContext` 可以直接接上。

---

## 四、memory_store.go

文件：`internal/todo/memory_store.go`

```go
package todo

import (
	"context"
	"sync"
)

type MemoryStore struct {
	mu     sync.RWMutex
	nextID int
	items  map[int]Todo
}

func NewMemoryStore() *MemoryStore {
	return &MemoryStore{
		nextID: 1,
		items:  make(map[int]Todo),
	}
}

func (s *MemoryStore) List(ctx context.Context) ([]Todo, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	items := make([]Todo, 0, len(s.items))
	for _, item := range s.items {
		items = append(items, item)
	}
	return items, nil
}

func (s *MemoryStore) Create(ctx context.Context, title string) (Todo, error) {
	s.mu.Lock()
	defer s.mu.Unlock()

	item := Todo{ID: s.nextID, Title: title}
	s.items[item.ID] = item
	s.nextID++
	return item, nil
}
```

继续补 `Get`、`Update`、`Delete`：

```go
func (s *MemoryStore) Get(ctx context.Context, id int) (Todo, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	item, ok := s.items[id]
	if !ok {
		return Todo{}, ErrNotFound
	}
	return item, nil
}

func (s *MemoryStore) Update(ctx context.Context, id int, input UpdateInput) (Todo, error) {
	s.mu.Lock()
	defer s.mu.Unlock()

	item, ok := s.items[id]
	if !ok {
		return Todo{}, ErrNotFound
	}
	if input.Title != nil {
		item.Title = *input.Title
	}
	if input.Done != nil {
		item.Done = *input.Done
	}
	s.items[id] = item
	return item, nil
}

func (s *MemoryStore) Delete(ctx context.Context, id int) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	if _, ok := s.items[id]; !ok {
		return ErrNotFound
	}
	delete(s.items, id)
	return nil
}
```

---

## 五、service.go

文件：`internal/todo/service.go`

```go
package todo

import (
	"context"
	"strings"
)

type Service struct {
	store Store
}

func NewService(store Store) *Service {
	return &Service{store: store}
}

func (s *Service) Create(ctx context.Context, title string) (Todo, error) {
	title = strings.TrimSpace(title)
	if title == "" {
		return Todo{}, ErrInvalidTitle
	}
	return s.store.Create(ctx, title)
}
```

Service 层负责业务规则。比如 title 不能为空，不应该散落在多个 Handler 中。

---

## 六、继续补齐 Service 方法

```go
func (s *Service) List(ctx context.Context) ([]Todo, error) {
	return s.store.List(ctx)
}

func (s *Service) Get(ctx context.Context, id int) (Todo, error) {
	return s.store.Get(ctx, id)
}

func (s *Service) Update(ctx context.Context, id int, input UpdateInput) (Todo, error) {
	if input.Title != nil {
		title := strings.TrimSpace(*input.Title)
		if title == "" {
			return Todo{}, ErrInvalidTitle
		}
		input.Title = &title
	}
	return s.store.Update(ctx, id, input)
}

func (s *Service) Delete(ctx context.Context, id int) error {
	return s.store.Delete(ctx, id)
}
```

---

## 七、本节检查点

完成后运行：

```bash
go test ./...
```

虽然还没有测试文件，但它能帮你确认所有包都能编译。

你应该能解释：

- 为什么内存 map 要加锁。
- 为什么 Store 方法带 context。
- 为什么 title 校验放在 Service。
- 为什么更新字段使用指针。
