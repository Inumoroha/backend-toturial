# 1. Cache-Aside 旁路缓存模式与读流程

第四阶段开始进入 Redis 在真实后端项目中最常见、也最容易踩坑的领域：缓存设计与一致性。

这一节先学习 Cache-Aside Pattern，也叫旁路缓存模式。它是业务系统使用 Redis 做缓存时最常见的模式之一。

学完这一节后，你应该能够：

- 理解 Cache-Aside Pattern 是什么。
- 掌握“先查缓存，未命中再查数据库，然后回写缓存”的读流程。
- 知道缓存命中、缓存未命中、数据库不存在分别怎么处理。
- 理解空值缓存为什么能防止缓存穿透。
- 写出文章详情缓存的基本读流程。

---

## 一、为什么需要缓存

假设一个文章详情接口非常热门。

如果每次请求都查数据库：

```text
用户请求 -> Go 服务 -> MySQL/PostgreSQL -> 返回文章
```

当访问量变大时，数据库压力会增加。

如果文章详情变化不频繁，可以把热点文章放进 Redis：

```text
用户请求 -> Go 服务 -> Redis
```

Redis 命中时，就不需要访问数据库。

缓存的目标不是替代数据库，而是减少重复查询、降低数据库压力、提升接口响应速度。

---

## 二、什么是 Cache-Aside Pattern

Cache-Aside Pattern 可以理解为：业务代码自己维护缓存。

读数据时：

```text
先查 Redis
命中则直接返回
未命中再查数据库
查到后写入 Redis
最后返回结果
```

写数据时：

```text
先更新数据库
再删除 Redis 缓存
```

这一节先讲读流程，下一节再讲写流程。

Cache-Aside 的核心思想是：数据库是权威数据源，Redis 是旁边的缓存层。

```text
          -> Redis 缓存
Go 服务
          -> 数据库权威数据
```

Redis 不主动同步数据库，数据库也不主动同步 Redis。同步逻辑由业务代码控制。

---

## 三、读流程总览

以文章详情为例。

key 设计：

```text
article:detail:{article_id}
```

请求文章 `1001` 时：

```text
1. 查询 Redis：GET article:detail:1001
2. 如果 Redis 命中，反序列化 JSON 并返回
3. 如果 Redis 未命中，查询数据库
4. 如果数据库查到文章，把文章序列化成 JSON 写入 Redis，并设置 TTL
5. 返回文章
```

伪代码：

```text
cacheKey = "article:detail:" + articleID

cached = redis.GET(cacheKey)
if cached exists:
    return JSON.parse(cached)

article = db.QueryArticle(articleID)
if article exists:
    redis.SET(cacheKey, JSON.stringify(article), EX, ttl)
    return article

return not found
```

这就是最经典的旁路缓存读流程。

---

## 四、缓存命中

缓存命中表示 Redis 里已经有数据。

命令示例：

```redis
GET article:detail:1001
```

返回：

```json
{"id":1001,"title":"Redis 缓存设计","content":"..."}
```

命中后，Go 服务通常会：

1. 反序列化 JSON。
2. 组装响应。
3. 直接返回给用户。

这时数据库完全没有参与。

缓存命中率越高，数据库压力越低。

---

## 五、缓存未命中

缓存未命中表示 Redis 没有这个 key。

```redis
GET article:detail:1001
```

返回：

```text
(nil)
```

未命中后，Go 服务需要查询数据库。

如果数据库查到文章：

```text
数据库 -> 返回文章详情
Go 服务 -> 写入 Redis
Go 服务 -> 返回文章详情
```

写入缓存：

```redis
SET article:detail:1001 '{"id":1001,"title":"Redis 缓存设计"}' EX 1800
```

这里的 `EX 1800` 表示 30 分钟后过期。

---

## 六、数据库也查不到怎么办

如果 Redis 未命中，数据库也查不到，这个数据是真的不存在。

比如用户访问不存在的文章 ID：

```text
GET /articles/999999
```

读流程：

```text
Redis 未命中
数据库也查不到
返回 404
```

这看起来没问题，但如果有人恶意大量请求不存在的 ID，每次都会穿过 Redis 打到数据库。

这就是缓存穿透的基础问题。

常见解决方式是空值缓存。

---

## 七、空值缓存

空值缓存是指：数据库查不到时，也向 Redis 写一个短 TTL 的空标记。

比如：

```redis
SET article:not_found:999999 1 EX 60
```

或者直接在原缓存 key 里写一个特殊值：

```redis
SET article:detail:999999 "__NULL__" EX 60
```

下次再请求这个不存在的文章，Redis 能命中空值标记，Go 服务直接返回 404，不再查数据库。

空值缓存 TTL 不宜太长。

原因是：这个数据未来可能被创建。如果空值缓存太久，用户可能在数据创建后仍然看到不存在。

常见 TTL：

```text
30 秒到 5 分钟
```

具体看业务。

---

## 八、缓存 TTL 怎么设置

文章详情缓存可以设置：

```text
30 分钟 + 0 到 10 分钟随机值
```

为什么加随机值？

如果大量 key 同一时间写入，并设置相同 TTL，它们可能同时过期，导致大量请求一起打到数据库。这就是缓存雪崩风险。

所以更好的策略是：

```text
基础 TTL + 随机偏移
```

例如：

```redis
SET article:detail:1001 '{...}' EX 2147
```

这里 `2147` 可能是程序计算出的随机 TTL。

---

## 九、Go 后端伪代码

文章详情读取流程可以写成：

```text
func GetArticle(ctx, id):
    key = ArticleDetailKey(id)

    cached = redis.Get(ctx, key)
    if cached == "__NULL__":
        return NotFound
    if cached exists:
        return DecodeArticle(cached)

    article = db.FindArticle(ctx, id)
    if article not found:
        redis.Set(ctx, key, "__NULL__", shortTTL)
        return NotFound

    redis.Set(ctx, key, EncodeArticle(article), ttlWithJitter)
    return article
```

注意几个点：

- Redis 未命中不是错误，是正常分支。
- 数据库查不到时可以写空值缓存。
- 缓存 TTL 最好加随机偏移。
- Redis 出错时，通常可以降级查数据库，但要记录日志。

---

## 十、Redis 出错怎么办

真实项目里 Redis 可能超时、连接失败或短暂不可用。

读缓存时，如果 Redis 出错，通常不要让整个接口直接失败。

更常见做法：

```text
Redis 查询失败 -> 记录日志 -> 继续查数据库 -> 返回结果
```

这叫降级。

但也要注意：如果 Redis 故障期间所有请求都打到数据库，数据库压力可能突然增大。

所以生产系统还需要限流、熔断、本地缓存等保护措施。

初学阶段先记住：缓存层失败，不应该轻易拖垮核心接口。

---

## 十一、读流程常见错误

### 1. 未命中后不回写缓存

这样每次都查数据库，缓存没有意义。

### 2. 缓存没有 TTL

缓存可能长期不刷新，用户看到旧数据。

### 3. 空值缓存 TTL 太长

数据创建后，用户仍然读到不存在。

### 4. Redis 错误和缓存未命中混在一起

缓存未命中是正常情况，Redis 连接失败是系统异常，代码里要区分。

### 5. 缓存 value 太大

文章详情可以缓存，但不要把整张文章表塞进一个 key。

---

## 十二、本节练习

请为文章详情接口设计读缓存流程：

1. 设计文章详情缓存 key。
2. 写出 Redis 命中时的处理逻辑。
3. 写出 Redis 未命中时查询数据库的逻辑。
4. 写出数据库查到后回写 Redis 的命令。
5. 写出数据库查不到时空值缓存的命令。
6. 给文章缓存设计 TTL。
7. 给空值缓存设计 TTL。
8. 思考：Redis 查询失败时，接口应该怎么降级？
9. 思考：为什么缓存未命中不等于系统错误？
10. 思考：为什么空值缓存不能设置太久？

---

## 十三、本节小结

这一节你学习了 Cache-Aside 的读流程。

你需要记住：

- Cache-Aside 是业务代码自己维护缓存。
- 读流程是：先查 Redis，未命中查数据库，查到后回写 Redis。
- 数据库是权威数据源，Redis 是缓存层。
- 数据库查不到时，可以写短 TTL 的空值缓存。
- 缓存 TTL 应结合业务，并尽量加入随机偏移。
- Redis 读取失败时，通常要考虑降级查数据库。

下一节我们学习写流程：为什么常见做法是更新数据库后删除缓存，而不是直接更新缓存。
