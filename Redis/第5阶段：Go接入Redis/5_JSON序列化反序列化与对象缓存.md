# 5. JSON 序列化、反序列化与对象缓存

Go 服务中使用 Redis 做缓存时，最常见的方式之一是：把结构体序列化成 JSON 字符串，写入 Redis String；读取时再反序列化回结构体。

这一节用文章详情缓存作为例子，讲清 JSON 缓存的基本写法和注意点。

学完这一节后，你应该能够：

- 定义适合缓存的 Go 结构体。
- 使用 `json.Marshal` 写入 Redis。
- 使用 `json.Unmarshal` 从 Redis 读取对象。
- 为对象缓存设置 TTL。
- 处理 JSON 反序列化失败。
- 理解缓存结构变更和 value 大小控制。

---

## 一、为什么常用 JSON 缓存对象

后端项目里经常有这样的数据：

```go
type Article struct {
    ID      int64  `json:"id"`
    Title   string `json:"title"`
    Content string `json:"content"`
    Author  string `json:"author"`
}
```

要缓存它，可以序列化成 JSON：

```json
{"id":1001,"title":"Redis 缓存","content":"...","author":"tom"}
```

然后写入 Redis：

```text
article:detail:1001 -> JSON 字符串
```

这种方式简单、直观、跨语言友好。

---

## 二、定义缓存结构体

示例：

```go
type Article struct {
    ID        int64     `json:"id"`
    Title     string    `json:"title"`
    Content   string    `json:"content"`
    AuthorID  int64     `json:"author_id"`
    CreatedAt time.Time `json:"created_at"`
}
```

有时你不想把数据库模型原样缓存。

可以单独定义缓存 DTO：

```go
type ArticleCacheDTO struct {
    ID       int64  `json:"id"`
    Title    string `json:"title"`
    Content  string `json:"content"`
    AuthorID int64  `json:"author_id"`
}
```

这样可以控制缓存里到底存哪些字段。

不要把不需要的敏感字段、超大字段都塞进 Redis。

---

## 三、生成缓存 key

建议集中封装 key 生成函数：

```go
func ArticleDetailKey(id int64) string {
    return fmt.Sprintf("article:detail:%d", id)
}
```

不要在代码各处手写：

```go
"article:detail:" + strconv.FormatInt(id, 10)
```

集中封装有几个好处：

- 命名统一。
- 修改前缀容易。
- 避免拼写错误。
- 测试更方便。

---

## 四、写入对象缓存

示例函数：

```go
func SetArticleCache(ctx context.Context, rdb *redis.Client, article ArticleCacheDTO) error {
    b, err := json.Marshal(article)
    if err != nil {
        return fmt.Errorf("marshal article cache: %w", err)
    }

    key := ArticleDetailKey(article.ID)
    ttl := 30 * time.Minute

    if err := rdb.Set(ctx, key, b, ttl).Err(); err != nil {
        return fmt.Errorf("set article cache: %w", err)
    }

    return nil
}
```

注意：`rdb.Set` 的 value 可以传 `[]byte`，也可以传 `string(b)`。

```go
rdb.Set(ctx, key, b, ttl)
```

Go 会把它作为二进制内容写入 Redis。对于 JSON 来说没问题。

---

## 五、读取对象缓存

读取并反序列化：

```go
func GetArticleCache(ctx context.Context, rdb *redis.Client, id int64) (*ArticleCacheDTO, error) {
    key := ArticleDetailKey(id)

    val, err := rdb.Get(ctx, key).Result()
    if err != nil {
        return nil, err
    }

    var article ArticleCacheDTO
    if err := json.Unmarshal([]byte(val), &article); err != nil {
        return nil, fmt.Errorf("unmarshal article cache: %w", err)
    }

    return &article, nil
}
```

这里没有处理 `redis.Nil`，是为了先看主流程。

后面错误处理篇会专门讲如何区分缓存未命中和真实错误。

---

## 六、缓存未命中时怎么处理

缓存未命中时，`Get` 返回 `redis.Nil`。

示例：

```go
article, err := GetArticleCache(ctx, rdb, id)
if err == nil {
    return article, nil
}

if errors.Is(err, redis.Nil) {
    // 查数据库，查到后回写缓存
}

// 其他错误：Redis 故障或 JSON 异常
```

注意：反序列化失败不是缓存未命中。

如果 JSON 格式坏了，可以记录日志并删除这个缓存，让下次从数据库重建。

---

## 七、TTL 随机化

对象缓存通常建议设置 TTL，并加入随机偏移。

```go
func ArticleCacheTTL() time.Duration {
    base := 30 * time.Minute
    jitter := time.Duration(rand.Intn(600)) * time.Second
    return base + jitter
}
```

写入：

```go
err := rdb.Set(ctx, key, b, ArticleCacheTTL()).Err()
```

注意：示例中使用 `math/rand` 时，Go 新版本默认种子行为已有变化；实际项目只需要确保能生成分散 TTL，不必为了缓存 TTL 使用强随机。

TTL 随机化的目标是防止大量 key 同时过期。

---

## 八、缓存字段兼容

缓存 JSON 结构可能随着代码升级变化。

比如旧缓存：

```json
{"id":1001,"title":"Redis"}
```

新结构体：

```go
type ArticleCacheDTO struct {
    ID       int64  `json:"id"`
    Title    string `json:"title"`
    AuthorID int64  `json:"author_id"`
}
```

旧 JSON 没有 `author_id`，反序列化后 `AuthorID` 是零值。

这可能没问题，也可能造成业务错误。

如果结构变化不兼容，可以给 key 加版本：

```go
func ArticleDetailKey(id int64) string {
    return fmt.Sprintf("article:detail:v2:%d", id)
}
```

这样旧缓存自然不会被新代码读取。

---

## 九、缓存 value 大小控制

不要把所有字段都塞进 Redis。

比如文章详情里可能有：

- 正文。
- 评论列表。
- 作者完整资料。
- 标签列表。
- 推荐文章列表。
- 各类统计。

如果全部塞进一个 JSON，value 会越来越大。

建议：

- 只缓存接口真正需要的字段。
- 大列表单独缓存。
- 评论列表和文章详情分开。
- 超大正文要评估是否适合缓存。
- 避免一个 key 保存无限增长数据。

缓存越大，网络传输和反序列化成本越高。

---

## 十、JSON 反序列化失败怎么办

可能原因：

- 旧版本缓存格式不兼容。
- 手动写入了错误 value。
- 程序 bug 写入了非 JSON。
- key 被其他模块误用。

处理建议：

```go
if err := json.Unmarshal([]byte(val), &article); err != nil {
    logger.Printf("bad article cache key=%s err=%v", key, err)
    _ = rdb.Del(ctx, key).Err()
    return nil, err
}
```

删除坏缓存后，下次请求可以从数据库重建。

但不要在所有场景里盲目删除，先确认 key 不会被其他结构使用。

---

## 十一、完整缓存读写示例

```go
type ArticleCache struct {
    rdb *redis.Client
}

func (c *ArticleCache) Set(ctx context.Context, article ArticleCacheDTO) error {
    b, err := json.Marshal(article)
    if err != nil {
        return fmt.Errorf("marshal article cache: %w", err)
    }

    key := ArticleDetailKey(article.ID)
    if err := c.rdb.Set(ctx, key, b, ArticleCacheTTL()).Err(); err != nil {
        return fmt.Errorf("set article cache: %w", err)
    }

    return nil
}

func (c *ArticleCache) Get(ctx context.Context, id int64) (*ArticleCacheDTO, error) {
    key := ArticleDetailKey(id)

    val, err := c.rdb.Get(ctx, key).Result()
    if err != nil {
        return nil, err
    }

    var article ArticleCacheDTO
    if err := json.Unmarshal([]byte(val), &article); err != nil {
        return nil, fmt.Errorf("unmarshal article cache: %w", err)
    }

    return &article, nil
}
```

这只是缓存模块，还没有数据库回源。完整文章缓存模块会在最后一篇实践中实现。

---

## 十二、常见错误

### 1. 缓存数据库模型全部字段

可能包含不需要的字段、敏感字段或大字段。

### 2. 忽略 Marshal 错误

虽然普通结构体很少失败，但错误仍然要处理。

### 3. 反序列化失败后继续使用零值对象

这很危险，可能返回错误数据。

### 4. 结构变化不考虑兼容

新代码读取旧缓存可能出现零值或语义错误。

### 5. 缓存 value 过大

Redis 快，不代表可以随便塞大 JSON。

---

## 十三、本节练习

请完成下面练习：

1. 定义 `ArticleCacheDTO` 结构体。
2. 写 `ArticleDetailKey(id int64)` 函数。
3. 写 `SetArticleCache`，使用 `json.Marshal` 和 `rdb.Set`。
4. 写 `GetArticleCache`，使用 `rdb.Get` 和 `json.Unmarshal`。
5. 给缓存设置 30 到 40 分钟随机 TTL。
6. 手动向 Redis 写入错误 JSON，观察反序列化错误。
7. 反序列化失败时删除坏缓存。
8. 思考哪些字段不应该放进文章详情缓存。
9. 给缓存 key 增加 `v2` 版本前缀。
10. 思考 String + JSON 和 Hash 在用户资料缓存中的取舍。

---

## 十四、本节小结

这一节你学习了 Go 中对象缓存的 JSON 写法。

你需要记住：

- String + JSON 是 Go 服务缓存对象的常见方式。
- 使用 `json.Marshal` 写入 Redis，使用 `json.Unmarshal` 读取对象。
- 缓存 key 要集中封装。
- 对象缓存要设置 TTL，并可加入随机偏移。
- 反序列化失败要记录日志，必要时删除坏缓存。
- 缓存结构变化时要考虑兼容或 key 版本。
- 不要把超大对象和无关字段都塞进一个 Redis value。

下一节我们会专门讲错误处理：如何区分 `redis.Nil`、Redis 系统错误、JSON 错误和 context 超时。
