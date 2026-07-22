# 11. Redis 缓存与登录失败限制

本节目标：给项目增加 Redis，用于缓存用户 profile，并限制登录失败次数。

---

## 一、安装 Redis 客户端

```bash
go get github.com/redis/go-redis/v9
```

---

## 二、初始化 Redis

`internal/database/redis.go`：

```go
package database

import "github.com/redis/go-redis/v9"

func NewRedis(addr, password string, db int) *redis.Client {
    return redis.NewClient(&redis.Options{
        Addr:     addr,
        Password: password,
        DB:       db,
    })
}
```

在 main 中：

```go
rdb := database.NewRedis(cfg.Redis.Addr, cfg.Redis.Password, cfg.Redis.DB)
```

---

## 三、缓存 profile

缓存 key：

```text
user:profile:{user_id}
```

查询流程：

```text
先查 Redis
命中则反序列化返回
未命中则查数据库
数据库查到后写入 Redis
```

示例：

```go
func (s *UserService) GetProfile(ctx context.Context, userID uint) (*model.User, error) {
    key := fmt.Sprintf("user:profile:%d", userID)

    cached, err := s.redis.Get(ctx, key).Result()
    if err == nil {
        var user model.User
        if json.Unmarshal([]byte(cached), &user) == nil {
            return &user, nil
        }
    }

    user, err := s.repo.GetByID(userID)
    if err != nil {
        return nil, err
    }

    bytes, _ := json.Marshal(user)
    _ = s.redis.Set(ctx, key, bytes, 10*time.Minute).Err()

    return user, nil
}
```

注意：Redis 失败不应该导致 profile 一定失败。缓存是优化，不是核心数据源。

---

## 四、登录失败限制

规则：

```text
同一邮箱 5 分钟内最多失败 5 次。
```

key：

```text
login:fail:{email}
```

记录失败：

```go
func (s *AuthService) RecordLoginFail(ctx context.Context, email string) {
    key := "login:fail:" + email
    count, _ := s.redis.Incr(ctx, key).Result()
    if count == 1 {
        s.redis.Expire(ctx, key, 5*time.Minute)
    }
}
```

检查限制：

```go
func (s *AuthService) CheckLoginLimit(ctx context.Context, email string) error {
    key := "login:fail:" + email
    count, err := s.redis.Get(ctx, key).Int()
    if err == redis.Nil {
        return nil
    }
    if err != nil {
        return nil
    }
    if count >= 5 {
        return ErrTooManyLoginAttempts
    }
    return nil
}
```

登录成功后删除：

```go
s.redis.Del(ctx, "login:fail:"+email)
```

---

## 五、为什么要传 context

handler 中：

```go
ctx := c.Request.Context()
```

传给 service：

```go
token, err := h.service.Login(ctx, input)
```

Redis 和数据库都应该尽量使用这个 ctx。请求取消时，后续操作可以尽快停止。

---

## 六、本节练习

完成：

- 初始化 Redis。
- profile 查询增加缓存。
- 登录失败记录次数。
- 连续失败超过 5 次返回 429。
- 登录成功清理失败次数。

---

## 七、本节验收

你应该能够回答：

- Redis 在这里解决了什么问题？
- 缓存失败时接口是否一定失败？
- 登录失败限制为什么适合用 Redis？
- 为什么 key 要包含邮箱？
- context 为什么要从 handler 传到 service？

---

## 八、Redis key 设计

建议统一 key 前缀：

```text
task-api:user:profile:{user_id}
task-api:login:fail:{email}
task-api:task:detail:{user_id}:{task_id}
```

为什么加 `task-api` 前缀？

如果同一个 Redis 被多个项目共用，前缀能避免 key 混在一起。

为什么任务详情 key 要包含 `user_id`？

任务是用户私有资源。即使 task_id 相同概率很低，也应该让 key 体现权限边界。

---

## 九、TTL 设计

推荐：

```text
用户 profile 缓存：10 分钟
任务详情缓存：5 分钟
登录失败次数：5 或 15 分钟
```

缓存类 key 必须有 TTL：

```go
redisClient.Set(ctx, key, data, 10*time.Minute)
```

登录失败计数也必须有 TTL，否则用户输错几次后可能长期无法登录。

```go
count, err := redisClient.Incr(ctx, key).Result()
if count == 1 {
    redisClient.Expire(ctx, key, 15*time.Minute)
}
```

---

## 十、缓存更新策略

profile 缓存读取：

```text
先读 Redis。
命中则返回。
未命中查数据库。
查到后写 Redis。
```

profile 更新：

```text
先更新数据库。
再删除 profile 缓存。
```

任务详情缓存也是类似：

```text
创建任务：可不写缓存。
查询详情：未命中后写缓存。
更新任务：删除详情缓存。
删除任务：删除详情缓存。
```

不要在更新数据库前先更新缓存。数据库失败时，会让缓存和数据库更容易不一致。

---

## 十一、Redis 降级策略

不同场景策略不同。

用户 profile 缓存失败：

```text
记录 warn 日志。
继续查数据库。
接口可以成功。
```

登录失败限制 Redis 失败：

```text
学习项目：记录 warn 日志，继续账号密码校验。
安全要求高的项目：可以拒绝登录或启用备用策略。
```

不要所有 Redis 错误都直接返回 `500`。先判断 Redis 在这个场景里是加速层，还是安全边界。

---

## 十二、验证步骤

启动 Redis：

```bash
docker compose up -d redis
```

查看：

```bash
docker compose exec redis redis-cli ping
```

验证 profile 缓存：

```bash
curl http://localhost:8080/api/v1/profile -H "Authorization: Bearer your-token"
docker compose exec redis redis-cli keys "task-api:user:profile:*"
```

验证登录失败：

```bash
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"demo@example.com","password":"wrong"}'
```

Redis 中查看：

```bash
docker compose exec redis redis-cli
GET task-api:login:fail:demo@example.com
TTL task-api:login:fail:demo@example.com
```

达到上限后，接口应该返回：

```text
HTTP 429
code = 42901
```

---

## 十三、常见问题

### 1. Redis 中没有 key

检查是否真的调用了缓存写入逻辑，key 前缀是否一致，是否连接错 Redis DB。

### 2. TTL 是 -1

说明 key 没有过期时间。缓存和登录失败计数都不应该长期无过期。

### 3. 登录成功后仍然被限制

检查登录成功后是否执行：

```go
redisClient.Del(ctx, key)
```

### 4. 更新 profile 后还是旧数据

检查更新成功后是否删除缓存。

---

## 十四、补充练习

请完成：

1. 给 Redis key 增加 `task-api` 前缀。
2. 给 profile 缓存设置 10 分钟 TTL。
3. 登录失败次数设置 15 分钟 TTL。
4. 登录成功后删除失败计数。
5. 更新 profile 后删除 profile 缓存。
6. Redis 出错时写 warn 日志。

---

## 十五、最终检查清单

- [ ] Redis 客户端只初始化一次。
- [ ] Redis 配置来自配置文件或环境变量。
- [ ] 缓存 key 命名清晰。
- [ ] 缓存 key 有 TTL。
- [ ] 登录失败 key 有 TTL。
- [ ] 达到失败次数后返回 `42901`。
- [ ] 登录成功后清理失败次数。
- [ ] 更新数据后删除相关缓存。
- [ ] Redis 失败时有明确降级策略。

完成这些后，Redis 才不是“加了个中间件”，而是真正服务于项目性能和安全边界。

