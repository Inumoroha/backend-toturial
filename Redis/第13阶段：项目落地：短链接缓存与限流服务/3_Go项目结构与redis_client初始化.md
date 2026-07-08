# 3. Go 项目结构与 redis client 初始化

这一节开始落到 Go 工程。

目标不是搭一个复杂框架，而是把项目结构、配置、Redis Client 和基础依赖整理清楚。

学完这一节后，你应该能够：

- 规划短链接服务的 Go 目录结构。
- 使用 `github.com/redis/go-redis/v9` 初始化客户端。
- 设置连接池、超时和认证信息。
- 在启动时检查 Redis 连通性。
- 为后续缓存、限流、计数功能准备基础代码。

---

## 一、推荐项目结构

可以使用下面结构：

```text
shortlink/
  cmd/
    api/
      main.go
  internal/
    config/
      config.go
    model/
      short_link.go
    repository/
      short_link_repository.go
    service/
      short_link_service.go
    cache/
      redis_client.go
      short_link_cache.go
    http/
      handler.go
      router.go
  go.mod
```

职责说明：

| 目录 | 职责 |
| --- | --- |
| `cmd/api` | 程序入口 |
| `config` | 配置读取 |
| `model` | 领域模型 |
| `repository` | 数据库访问 |
| `service` | 业务逻辑 |
| `cache` | Redis 访问 |
| `http` | 路由和 handler |

学习项目可以简化，但不要把所有代码都写进 `main.go`。

---

## 二、安装 go-redis

```bash
go get github.com/redis/go-redis/v9
```

使用时需要传入 `context.Context`：

```go
import "github.com/redis/go-redis/v9"
```

go-redis v9 是目前 Go 项目中常用的 Redis 客户端之一。

---

## 三、配置结构

配置可以先用环境变量。

```go
type RedisConfig struct {
    Addr         string
    Username     string
    Password     string
    DB           int
    PoolSize     int
    MinIdleConns int
}
```

示例环境变量：

```text
REDIS_ADDR=127.0.0.1:6379
REDIS_USERNAME=shortlink_app
REDIS_PASSWORD=app-password
REDIS_DB=0
```

不要把密码写死在代码里。

---

## 四、初始化 Redis Client

示例：

```go
func NewRedisClient(cfg RedisConfig) *redis.Client {
    return redis.NewClient(&redis.Options{
        Addr:         cfg.Addr,
        Username:     cfg.Username,
        Password:     cfg.Password,
        DB:           cfg.DB,
        PoolSize:     cfg.PoolSize,
        MinIdleConns: cfg.MinIdleConns,
        DialTimeout:  2 * time.Second,
        ReadTimeout:  500 * time.Millisecond,
        WriteTimeout: 500 * time.Millisecond,
    })
}
```

超时不要无限大。

Redis 是加速组件，调用 Redis 卡住会拖垮业务接口。

---

## 五、启动时 Ping

启动时可以做一次连通性检查：

```go
func CheckRedis(ctx context.Context, rdb *redis.Client) error {
    return rdb.Ping(ctx).Err()
}
```

入口示例：

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

if err := CheckRedis(ctx, rdb); err != nil {
    log.Printf("redis ping failed: %v", err)
}
```

是否因为 Redis 不可用而拒绝启动，要看业务。

如果短链接详情可以回源数据库，Redis 不可用时服务仍然可以启动，但要记录告警。

如果 Redis 承担强依赖能力，比如验证码限流，可能要更保守。

---

## 六、Context 超时

每次 Redis 操作都应该带 context。

示例：

```go
ctx, cancel := context.WithTimeout(r.Context(), 300*time.Millisecond)
defer cancel()

val, err := rdb.Get(ctx, key).Result()
```

不要在请求链路里使用没有超时的 `context.Background()`。

推荐做法：

- HTTP 请求上下文作为父 context。
- Redis 操作使用短超时。
- 数据库操作也设置合理超时。

---

## 七、缓存层封装

不要在 handler 里到处拼 Redis key。

可以封装一个 Cache：

```go
type ShortLinkCache struct {
    rdb *redis.Client
}

func NewShortLinkCache(rdb *redis.Client) *ShortLinkCache {
    return &ShortLinkCache{rdb: rdb}
}
```

后续方法：

```go
func (c *ShortLinkCache) GetDetail(ctx context.Context, code string) (*CachedShortLink, error)
func (c *ShortLinkCache) SetDetail(ctx context.Context, link CachedShortLink, ttl time.Duration) error
func (c *ShortLinkCache) IncrVisit(ctx context.Context, code string) error
func (c *ShortLinkCache) HitCreateLimit(ctx context.Context, userID int64) (bool, error)
```

缓存层负责 Redis 命令。

服务层负责业务决策。

---

## 八、key 构造函数

统一写 key 构造函数：

```go
func detailKey(code string) string {
    return "shortlink:detail:" + code
}

func visitCountKey(code string) string {
    return "shortlink:visit_count:" + code
}

func createLimitKey(userID int64) string {
    return fmt.Sprintf("shortlink:create_limit:%d", userID)
}

func notFoundKey(code string) string {
    return "shortlink:not_found:" + code
}
```

不要在多个地方手写同一个 key。

否则后期改名会漏。

---

## 九、错误处理原则

Redis 错误分为几类：

| 错误 | 处理 |
| --- | --- |
| `redis.Nil` | key 不存在，属于正常情况 |
| 超时 | 记录日志，走降级 |
| 连接失败 | 记录日志，走数据库或拒绝 |
| 序列化失败 | 代码或数据问题，需要修复 |

示例：

```go
if errors.Is(err, redis.Nil) {
    return nil, nil
}
if err != nil {
    return nil, fmt.Errorf("get shortlink cache: %w", err)
}
```

不要把 `redis.Nil` 当成系统错误。

---

## 十、日志建议

至少记录：

- Redis 连接失败。
- Redis 超时。
- 缓存反序列化失败。
- 限流拒绝。
- 数据库回源失败。
- 更新后删除缓存失败。

日志要带关键字段：

```text
code=aB9xK2
user_id=1001
key=shortlink:detail:aB9xK2
err=...
```

不要在日志里打印 Redis 密码。

---

## 十一、常见错误

### 1. 每个请求创建一个 Redis Client

Redis Client 应该复用。

每个请求新建连接池会造成连接数暴涨。

### 2. 没有设置超时

Redis 一旦变慢，会拖慢整个 HTTP 请求。

### 3. handler 里直接写 Redis 命令

业务逻辑会散落各处，后面很难维护。

### 4. 忽略 `redis.Nil`

key 不存在不是故障。

### 5. 配置写死

地址、用户名、密码、DB、连接池都应该通过配置管理。

---

## 十二、本节练习

请完成下面练习：

1. 创建短链接项目目录结构。
2. 安装 `github.com/redis/go-redis/v9`。
3. 定义 Redis 配置结构。
4. 初始化 Redis Client。
5. 写启动时 Ping 检查。
6. 写 4 个 Redis key 构造函数。
7. 封装 `ShortLinkCache`。

---

## 十三、本节小结

这一节完成了 Go 项目的 Redis 基础设施。

你需要记住：

- Redis Client 要全局复用。
- 所有 Redis 操作都要带 context 和超时。
- key 构造要集中管理。
- 缓存访问要封装在独立层里。
- Redis 错误要区分 `redis.Nil` 和真正故障。

下一节我们实现短链接详情缓存：Cache-Aside 与空值缓存。

