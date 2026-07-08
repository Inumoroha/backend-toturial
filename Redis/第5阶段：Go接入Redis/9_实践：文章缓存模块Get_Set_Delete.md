# 9. 实践：文章缓存模块 Get、Set、Delete

前面几节分别学习了 `go-redis/v9` 初始化、连接池、context、常用命令、JSON、错误处理、Pipeline 和 Lua。

这一节把它们组合起来，写一个文章缓存模块。

目标是实现：

- `GetArticle(ctx, id)`：优先查 Redis，未命中查数据库，然后回写缓存。
- `SetArticleCache(ctx, article)`：将文章序列化为 JSON 后写入 Redis。
- `DeleteArticleCache(ctx, id)`：更新文章后删除缓存。
- Redis 操作使用合理超时。
- Redis 故障时允许降级到数据库。

---

## 一、模块职责设计

先拆清楚职责。

```text
ArticleRepository：负责数据库读写
ArticleCache：负责 Redis 缓存读写
ArticleService：负责业务流程，组织缓存和数据库
```

不要把 Redis 命令直接写在 HTTP Handler 里。

推荐结构：

```text
internal/article/
  model.go
  repository.go
  cache.go
  service.go
```

这样每层职责清晰，也更容易测试。

---

## 二、定义 Article 模型

```go
package article

import "time"

type Article struct {
    ID        int64     `json:"id"`
    Title     string    `json:"title"`
    Content   string    `json:"content"`
    AuthorID  int64     `json:"author_id"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
}
```

真实项目中，数据库模型和缓存 DTO 可以分开。

本教程为了聚焦 Redis，直接使用一个结构体。

---

## 三、定义数据库仓储接口

为了让示例不绑定具体 MySQL 或 PostgreSQL，先定义接口：

```go
package article

import "context"

var ErrArticleNotFound = errors.New("article not found")

type Repository interface {
    FindByID(ctx context.Context, id int64) (*Article, error)
    Update(ctx context.Context, article *Article) error
}
```

`FindByID` 查不到文章时返回 `ErrArticleNotFound`。

后面 Service 层会根据这个错误决定是否写空值缓存。

---

## 四、定义缓存 key

```go
package article

import "fmt"

func ArticleDetailKey(id int64) string {
    return fmt.Sprintf("article:detail:%d", id)
}
```

如果你想支持版本化：

```go
func ArticleDetailKey(id int64) string {
    return fmt.Sprintf("article:detail:v1:%d", id)
}
```

key 生成函数要集中管理，不要散落在代码各处。

---

## 五、定义 ArticleCache

```go
package article

import (
    "context"
    "encoding/json"
    "errors"
    "fmt"
    "log"
    "math/rand"
    "time"

    "github.com/redis/go-redis/v9"
)

type ArticleCache struct {
    rdb    *redis.Client
    logger *log.Logger
}

func NewArticleCache(rdb *redis.Client, logger *log.Logger) *ArticleCache {
    return &ArticleCache{rdb: rdb, logger: logger}
}
```

缓存模块只关心 Redis，不直接访问数据库。

---

## 六、缓存 TTL

```go
func articleCacheTTL() time.Duration {
    base := 30 * time.Minute
    jitter := time.Duration(rand.Intn(600)) * time.Second
    return base + jitter
}
```

表示 30 到 40 分钟随机 TTL。

空值缓存 TTL：

```go
func articleNullCacheTTL() time.Duration {
    return 1 * time.Minute
}
```

空值缓存不要太长，避免文章创建后还被判断为不存在。

---

## 七、SetArticleCache

```go
func (c *ArticleCache) SetArticleCache(ctx context.Context, article *Article) error {
    redisCtx, cancel := context.WithTimeout(ctx, 100*time.Millisecond)
    defer cancel()

    b, err := json.Marshal(article)
    if err != nil {
        return fmt.Errorf("marshal article cache: %w", err)
    }

    key := ArticleDetailKey(article.ID)
    if err := c.rdb.Set(redisCtx, key, b, articleCacheTTL()).Err(); err != nil {
        return fmt.Errorf("set article cache: %w", err)
    }

    return nil
}
```

注意：

- 使用短超时保护 Redis 写操作。
- JSON 序列化失败要返回错误。
- Redis 写入失败要返回给调用方，让 Service 层决定是否忽略或记录。

---

## 八、GetArticleCache

```go
func (c *ArticleCache) GetArticleCache(ctx context.Context, id int64) (*Article, error) {
    redisCtx, cancel := context.WithTimeout(ctx, 50*time.Millisecond)
    defer cancel()

    key := ArticleDetailKey(id)
    val, err := c.rdb.Get(redisCtx, key).Result()
    if err != nil {
        return nil, err
    }

    if val == "__NULL__" {
        return nil, ErrArticleNotFound
    }

    var article Article
    if err := json.Unmarshal([]byte(val), &article); err != nil {
        _ = c.rdb.Del(context.Background(), key).Err()
        return nil, fmt.Errorf("unmarshal article cache: %w", err)
    }

    return &article, nil
}
```

这里处理了空值缓存：

```text
__NULL__ -> 文章不存在
```

反序列化失败时删除坏缓存。

注意：示例里删除坏缓存用了 `context.Background()`，真实项目中可以使用带短超时的新 context，并记录删除失败日志。

---

## 九、SetNullArticleCache

数据库也查不到文章时，写短 TTL 空值缓存：

```go
func (c *ArticleCache) SetNullArticleCache(ctx context.Context, id int64) error {
    redisCtx, cancel := context.WithTimeout(ctx, 100*time.Millisecond)
    defer cancel()

    key := ArticleDetailKey(id)
    if err := c.rdb.Set(redisCtx, key, "__NULL__", articleNullCacheTTL()).Err(); err != nil {
        return fmt.Errorf("set null article cache: %w", err)
    }

    return nil
}
```

空值缓存用于防止缓存穿透。

---

## 十、DeleteArticleCache

更新文章后删除缓存：

```go
func (c *ArticleCache) DeleteArticleCache(ctx context.Context, id int64) error {
    redisCtx, cancel := context.WithTimeout(ctx, 100*time.Millisecond)
    defer cancel()

    key := ArticleDetailKey(id)
    if err := c.rdb.Del(redisCtx, key).Err(); err != nil {
        return fmt.Errorf("delete article cache: %w", err)
    }

    return nil
}
```

删除缓存失败需要关注，因为旧缓存可能继续存在。

Service 层可以记录日志，并投递重试任务。

---

## 十一、ArticleService

```go
type ArticleService struct {
    repo   Repository
    cache  *ArticleCache
    logger *log.Logger
}

func NewArticleService(repo Repository, cache *ArticleCache, logger *log.Logger) *ArticleService {
    return &ArticleService{repo: repo, cache: cache, logger: logger}
}
```

Service 层负责组织流程：先缓存，后数据库。

---

## 十二、GetArticle：优先查缓存

```go
func (s *ArticleService) GetArticle(ctx context.Context, id int64) (*Article, error) {
    article, err := s.cache.GetArticleCache(ctx, id)
    if err == nil {
        return article, nil
    }

    if errors.Is(err, ErrArticleNotFound) {
        return nil, ErrArticleNotFound
    }

    if !errors.Is(err, redis.Nil) {
        s.logger.Printf("get article cache failed id=%d err=%v", id, err)
    }

    article, err = s.repo.FindByID(ctx, id)
    if err != nil {
        if errors.Is(err, ErrArticleNotFound) {
            if cacheErr := s.cache.SetNullArticleCache(ctx, id); cacheErr != nil {
                s.logger.Printf("set null article cache failed id=%d err=%v", id, cacheErr)
            }
        }
        return nil, err
    }

    if err := s.cache.SetArticleCache(ctx, article); err != nil {
        s.logger.Printf("set article cache failed id=%d err=%v", id, err)
    }

    return article, nil
}
```

流程：

```text
查 Redis
命中：返回
命中空值：返回不存在
未命中：查数据库
数据库查不到：写空值缓存
数据库查到：写正常缓存
Redis 系统错误：记录日志，降级查数据库
```

---

## 十三、UpdateArticle：更新数据库后删除缓存

```go
func (s *ArticleService) UpdateArticle(ctx context.Context, article *Article) error {
    if err := s.repo.Update(ctx, article); err != nil {
        return err
    }

    if err := s.cache.DeleteArticleCache(ctx, article.ID); err != nil {
        s.logger.Printf("delete article cache failed id=%d err=%v", article.ID, err)
        // 真实项目可以投递重试任务
    }

    return nil
}
```

这里遵循第四阶段讲过的写流程：

```text
先更新数据库
再删除缓存
```

删除缓存失败时，至少要记录日志。

更完整的生产方案可以加入：

- 重试。
- 延迟双删。
- 消息队列补偿。
- TTL 兜底。

---

## 十四、Redis 故障时如何降级

在 `GetArticle` 中，如果 Redis 读取失败：

```go
if !errors.Is(err, redis.Nil) {
    logger.Printf("get cache failed")
}
article, err := repo.FindByID(ctx, id)
```

这表示 Redis 故障时允许查数据库。

但要注意：如果 Redis 故障期间流量很大，所有请求都打数据库，数据库可能被压垮。

生产系统还应该有：

- 限流。
- 熔断。
- 本地缓存。
- 监控告警。
- 降级策略。

教程里先实现最基本的降级。

---

## 十五、context 超时建议

示例中使用：

```text
缓存 GET：50ms
缓存 SET/DEL：100ms
```

这只是示例值。

实际项目要根据：

- 接口总超时。
- Redis 部署位置。
- 网络延迟。
- Redis 命令耗时。
- 是否允许降级。

来调整。

原则：缓存操作不要比主请求更长，也不要在 Redis 抖动时拖死接口。

---

## 十六、可以继续改进的点

这个模块是教学版，还可以继续增强：

- 使用自定义 `ErrCacheMiss` 隔离 `go-redis`。
- 对坏缓存删除使用短超时 context。
- 加入指标统计：命中、未命中、Redis 错误。
- 对热点文章使用互斥重建或逻辑过期。
- 对删除缓存失败投递重试任务。
- 使用 Pipeline 批量获取文章列表缓存。
- 加入单元测试，mock Redis 和 Repository。

这些会让模块更接近生产级代码。

---

## 十七、常见错误

### 1. Handler 里直接写 Redis 命令

会让业务逻辑混乱，也不容易测试。

### 2. 缓存未命中直接返回错误

`redis.Nil` 应该触发数据库回源。

### 3. Redis 错误完全吞掉

至少要打日志，否则故障不可见。

### 4. 更新数据库后忘记删缓存

用户可能长时间读到旧数据。

### 5. 删除缓存失败不处理

旧缓存可能继续存在，应该记录并考虑补偿。

### 6. 不设置缓存 TTL

删除失败后没有兜底过期。

---

## 十八、本节练习

请完成下面练习：

1. 定义 `Article` 结构体。
2. 定义 `Repository` 接口。
3. 实现 `ArticleDetailKey`。
4. 实现 `ArticleCache.SetArticleCache`。
5. 实现 `ArticleCache.GetArticleCache`。
6. 实现 `ArticleCache.DeleteArticleCache`。
7. 实现空值缓存。
8. 实现 `ArticleService.GetArticle`。
9. 实现 `ArticleService.UpdateArticle`。
10. 给 Redis 操作加短超时，并在 Redis 错误时降级查数据库。

---

## 十九、本节小结

这一节你完成了第五阶段的实践闭环。

你需要记住：

- Redis 操作应该封装在缓存模块中。
- Service 层负责组织 Cache-Aside 流程。
- `GetArticle` 优先查 Redis，未命中查数据库，查到后回写缓存。
- 数据库查不到时可以写短 TTL 空值缓存。
- `SetArticleCache` 使用 JSON 序列化，并设置 TTL。
- `DeleteArticleCache` 用于更新数据库后删除缓存。
- Redis 错误要记录日志，缓存场景通常允许降级到数据库。
- `context` 超时是 Go 接入 Redis 的基本功。

学完第五阶段，你已经能在 Go 服务中比较稳定、清晰地操作 Redis，并把 Redis 放进真实请求链路中。
