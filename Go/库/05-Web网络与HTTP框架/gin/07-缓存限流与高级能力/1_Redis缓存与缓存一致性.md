# 1. Redis 缓存与缓存一致性

本节目标：理解缓存适合解决什么问题，以及在 Gin 项目中使用缓存时必须考虑的一致性问题。

在很多后端项目里，Redis 是最常见的中间件之一。但初学者很容易把 Redis 理解成“更快的数据库”，这是不准确的。Redis 更常见的定位是缓存、计数器、短期状态存储和分布式辅助组件。

---

## 一、什么时候需要缓存

缓存适合这些场景：

- 数据读取频繁。
- 数据变化不频繁。
- 查询成本较高。
- 对短时间旧数据有一定容忍度。

例如：

```text
用户公开资料
文章详情
商品分类
系统配置
热门列表
```

不适合随便缓存的数据：

```text
余额
库存扣减
订单支付状态
权限实时变更
```

这些数据对一致性要求高，缓存策略要更谨慎。

---

## 二、缓存基本流程

最常见的是 Cache Aside 模式：

```text
读取数据：
1. 先查 Redis
2. Redis 命中，直接返回
3. Redis 未命中，查数据库
4. 数据库查到后，写入 Redis
5. 返回结果

修改数据：
1. 先修改数据库
2. 再删除缓存
```

为什么修改后通常是删除缓存，而不是直接更新缓存？

因为数据库更新和缓存更新不是一个原子操作。如果更新缓存失败，可能导致缓存和数据库不一致。删除缓存后，下次读取会重新从数据库加载最新数据。

---

## 三、缓存 key 如何设计

缓存 key 要可读、稳定、可定位。

用户详情可以设计为：

```text
user:detail:1
user:detail:2
```

不要设计成：

```text
u1
data_1
cache_user
```

好的 key 应该包含：

- 业务模块。
- 数据类型。
- 唯一标识。

示例：

```go
func userDetailKey(userID uint) string {
    return fmt.Sprintf("user:detail:%d", userID)
}
```

如果项目较大，可以增加项目前缀：

```text
gin-user-api:user:detail:1
```

---

## 四、缓存过期时间

缓存一般要设置过期时间：

```go
redisClient.Set(ctx, key, value, 10*time.Minute)
```

不设置过期时间的风险：

- 脏数据长期存在。
- Redis 内存持续增长。
- 删除缓存遗漏时难以恢复。

过期时间不要所有 key 完全一样。高并发项目中，如果大量 key 同时过期，可能引发缓存雪崩。可以加随机抖动：

```go
ttl := 10*time.Minute + time.Duration(rand.Intn(60))*time.Second
```

学习阶段先掌握固定 TTL，再理解随机 TTL 的意义。

---

## 五、三类常见缓存问题

### 1. 缓存穿透

请求的数据数据库中也不存在，例如一直请求不存在的用户 ID：

```text
GET /api/v1/users/999999999
```

每次 Redis 都查不到，每次都打到数据库。

常见处理：

- 参数校验，拒绝明显非法 ID。
- 缓存空结果，设置较短 TTL。
- 布隆过滤器，大型项目中使用。

学习项目可以使用“缓存空结果”：

```text
user:detail:999 = "__nil__" TTL 1 minute
```

### 2. 缓存击穿

某个热点 key 过期，大量请求同时查数据库。

常见处理：

- 热点数据更长 TTL。
- 互斥锁。
- 后台刷新缓存。

学习项目先理解概念即可，不必一开始实现分布式锁。

### 3. 缓存雪崩

大量 key 同时过期，导致数据库压力瞬间升高。

常见处理：

- TTL 加随机抖动。
- 分批预热。
- 限流降级。

---

## 六、缓存一致性策略

用户详情读取：

```text
读 Redis -> 未命中读 DB -> 写 Redis -> 返回
```

用户资料更新：

```text
更新 DB -> 删除 user:detail:{id}
```

如果删除缓存失败怎么办？

学习阶段可以：

- 记录 error 日志。
- 返回业务成功。
- 依靠较短 TTL 最终恢复。

生产中可以进一步：

- 重试删除。
- 写消息队列异步删除。
- 使用延迟双删。

不要一上来就堆复杂方案。先把最基础的 Cache Aside 写对。

---

## 七、JSON 序列化缓存

Go 结构体写入 Redis 前通常转 JSON：

```go
data, err := json.Marshal(user)
if err != nil {
    return err
}

err = redisClient.Set(ctx, key, data, 10*time.Minute).Err()
```

读取时：

```go
value, err := redisClient.Get(ctx, key).Result()
if err == redis.Nil {
    return nil, nil
}
if err != nil {
    return nil, err
}

var user UserDTO
if err := json.Unmarshal([]byte(value), &user); err != nil {
    return nil, err
}
```

缓存 DTO 通常比缓存数据库 model 更合适，因为 DTO 更接近接口响应，也更少包含敏感字段。

---

## 八、缓存放在哪一层

推荐放在 service 层或专门的 cache 层，由 service 组合使用：

```text
handler -> service -> cache
                 -> repository
```

handler 不应该直接操作 Redis。handler 应该只关心 HTTP 请求和响应。

repository 通常只关心数据库。把 Redis 混进 repository 会让职责变模糊。

---

## 九、常见问题

### 1. Redis 缓存里能不能放完整用户模型

不建议。数据库模型可能包含密码哈希、内部状态等字段。缓存接口响应需要的数据即可。

### 2. 数据更新后为什么还读到旧数据

检查更新逻辑是否删除了缓存 key。也检查 key 生成函数是否一致。

### 3. Redis 挂了接口是不是必须失败

看业务。用户详情这类缓存场景，Redis 失败时可以降级查数据库并记录日志。登录限流这类安全场景，要根据安全策略决定。

---

## 十、练习

请设计以下缓存 key：

1. 用户详情。
2. 用户列表第一页。
3. 邮箱验证码。
4. 登录失败次数。
5. 文章详情。

并写出每个 key 建议的 TTL。

---

## 十一、验收标准

完成本节后，你应该能回答：

- 什么数据适合缓存。
- Cache Aside 的读取和更新流程是什么。
- 修改数据后为什么通常删除缓存。
- 缓存穿透、击穿、雪崩分别是什么。
- Redis 操作应该放在 Gin 项目的哪一层。

理解这些，再进入 Redis 初始化实践。

