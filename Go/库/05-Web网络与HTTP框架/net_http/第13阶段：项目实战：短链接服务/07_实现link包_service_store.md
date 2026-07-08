# 7. 实现 link 包：service 与 store

本节目标：把短链接核心业务放进 `internal/link` 包中，先实现内存版本。

---

## 一、推荐目录

```text
shortener/
  cmd/
    server/
      main.go
  internal/
    link/
      model.go
      errors.go
      store.go
      memory_store.go
      service.go
      code.go
    httpapi/
    middleware/
  go.mod
```

---

## 二、model.go

```go
package link

import "time"

type Link struct {
	Code        string    `json:"code"`
	OriginalURL string    `json:"original_url"`
	VisitCount  int64     `json:"visit_count"`
	CreatedAt   time.Time `json:"created_at"`
}
```

---

## 三、errors.go

```go
package link

import "errors"

var ErrNotFound = errors.New("link not found")
var ErrConflict = errors.New("code already exists")
var ErrInvalidURL = errors.New("invalid url")
```

这些错误由 Service 或 Store 返回，HTTP 层再映射成状态码。

---

## 四、store.go

```go
package link

import "context"

type Store interface {
	Create(ctx context.Context, item Link) (Link, error)
	GetByCode(ctx context.Context, code string) (Link, error)
	IncrementVisit(ctx context.Context, code string) error
	List(ctx context.Context) ([]Link, error)
}
```

---

## 五、memory_store.go

```go
type MemoryStore struct {
	mu    sync.RWMutex
	items map[string]Link
}

func NewMemoryStore() *MemoryStore {
	return &MemoryStore{items: make(map[string]Link)}
}
```

创建时在锁内判断冲突：

```go
func (s *MemoryStore) Create(ctx context.Context, item Link) (Link, error) {
	s.mu.Lock()
	defer s.mu.Unlock()

	if _, ok := s.items[item.Code]; ok {
		return Link{}, ErrConflict
	}
	s.items[item.Code] = item
	return item, nil
}
```

访问计数：

```go
func (s *MemoryStore) IncrementVisit(ctx context.Context, code string) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	item, ok := s.items[code]
	if !ok {
		return ErrNotFound
	}
	item.VisitCount++
	s.items[code] = item
	return nil
}
```

---

## 六、service.go

Service 负责：

- 校验 URL。
- 生成短码。
- 冲突后重试。
- 调用 Store。

```go
type Service struct {
	store Store
}

func NewService(store Store) *Service {
	return &Service{store: store}
}
```

创建逻辑：

```go
func (s *Service) Create(ctx context.Context, rawURL string) (Link, error) {
	if err := validateURL(rawURL); err != nil {
		return Link{}, ErrInvalidURL
	}

	for i := 0; i < 5; i++ {
		item := Link{
			Code:        GenerateCode(6),
			OriginalURL: rawURL,
			CreatedAt:   time.Now(),
		}
		created, err := s.store.Create(ctx, item)
		if err == nil {
			return created, nil
		}
		if !errors.Is(err, ErrConflict) {
			return Link{}, err
		}
	}

	return Link{}, ErrConflict
}
```

---

## 七、本节检查点

你应该能解释：

- Store 为什么要抽象成接口。
- 为什么短码冲突要由 Store 兜底。
- 为什么访问计数更新要加写锁。
- Service 为什么负责 URL 校验和冲突重试。
