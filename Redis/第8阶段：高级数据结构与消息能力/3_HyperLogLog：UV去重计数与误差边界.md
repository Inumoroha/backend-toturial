# 3. HyperLogLog：UV 去重计数与误差边界

HyperLogLog 是 Redis 用来做去重计数的结构。

它最大的特点是：用很小内存估算大量元素的去重数量，但结果有少量误差。

学完这一节后，你应该能够：

- 理解 HyperLogLog 适合什么场景。
- 使用 `PFADD`、`PFCOUNT`、`PFMERGE`。
- 用 HyperLogLog 统计页面 UV。
- 理解它和 Set 的区别。
- 知道误差和不能取明细的限制。

---

## 一、UV 是什么

PV：

```text
Page View，页面访问次数。
```

同一个用户访问 10 次，PV 加 10。

UV：

```text
Unique Visitor，独立访问用户数。
```

同一个用户访问 10 次，UV 只算 1。

PV 可以用计数器：

```redis
INCR page:pv:home:20260705
```

UV 需要去重计数。

---

## 二、用 Set 统计 UV 的问题

Set 做法：

```redis
SADD page:uv:set:home:20260705 user:1001
SCARD page:uv:set:home:20260705
```

优点：

- 精确。
- 可以知道具体有哪些用户。

缺点：

- 用户量大时内存占用高。
- 每天、每页面都保存完整用户集合，成本很高。

如果你只需要 UV 数量，不需要成员列表，HyperLogLog 更合适。

---

## 三、HyperLogLog 基本命令

添加元素：

```redis
PFADD page:uv:home:20260705 user:1001
```

统计去重数量：

```redis
PFCOUNT page:uv:home:20260705
```

合并多个 HyperLogLog：

```redis
PFMERGE page:uv:home:week page:uv:home:20260701 page:uv:home:20260702
```

注意命令前缀是 `PF`。

---

## 四、统计页面 UV

key 设计：

```text
page:uv:{page_id}:{yyyyMMdd}
```

示例：

```text
page:uv:article:1001:20260705
```

访问时：

```redis
PFADD page:uv:article:1001:20260705 user:2001
```

查询：

```redis
PFCOUNT page:uv:article:1001:20260705
```

HyperLogLog 会自动去重。

同一个 `user:2001` 添加多次，估算 UV 不会按次数增加。

---

## 五、Go 实现 UV 统计

```go
func PageUVKey(pageID string, now time.Time) string {
    return fmt.Sprintf("page:uv:%s:%s", pageID, now.Format("20060102"))
}
```

记录访问：

```go
func AddPageUV(ctx context.Context, rdb *redis.Client, pageID string, userID int64, now time.Time) error {
    key := PageUVKey(pageID, now)
    member := fmt.Sprintf("user:%d", userID)

    pipe := rdb.Pipeline()
    pipe.PFAdd(ctx, key, member)
    pipe.Expire(ctx, key, 90*24*time.Hour)
    _, err := pipe.Exec(ctx)
    return err
}
```

查询：

```go
func CountPageUV(ctx context.Context, rdb *redis.Client, pageID string, day time.Time) (int64, error) {
    key := PageUVKey(pageID, day)
    return rdb.PFCount(ctx, key).Result()
}
```

---

## 六、统计周 UV

可以把每天的 HyperLogLog 合并：

```redis
PFMERGE page:uv:article:1001:week:2026W27 \
  page:uv:article:1001:20260701 \
  page:uv:article:1001:20260702 \
  page:uv:article:1001:20260703
```

然后：

```redis
PFCOUNT page:uv:article:1001:week:2026W27
```

Go 中可以：

```go
func MergeUV(ctx context.Context, rdb *redis.Client, dest string, sources ...string) error {
    return rdb.PFMerge(ctx, dest, sources...).Err()
}
```

合并后的 key 也要设置 TTL。

---

## 七、误差边界

HyperLogLog 是概率算法。

它的结果不是完全精确的。

Redis HyperLogLog 的标准误差大约在 1% 左右。

这意味着：

```text
真实 UV = 1,000,000
估算结果可能是 990,000 或 1,010,000 附近
```

具体结果会有波动。

如果业务必须精确，比如发奖、结算、计费，就不要只用 HyperLogLog。

---

## 八、HyperLogLog 不能取出成员

这是它最重要的限制。

你可以：

```redis
PFCOUNT page:uv:home:20260705
```

但不能：

```text
列出所有访问过的 user_id
```

如果需要明细，用 Set、数据库日志或数据仓库。

HyperLogLog 只回答：

```text
大概有多少个不同元素？
```

---

## 九、什么时候适合 HyperLogLog

适合：

- 页面 UV。
- 活动 UV。
- 搜索 UV。
- 接口独立调用用户数。
- 大盘趋势统计。
- 允许少量误差的报表。

不适合：

- 计费。
- 发奖。
- 权限判断。
- 精确人数。
- 需要用户明细。
- 小规模但要求完全准确的数据。

---

## 十、PV 和 UV 一起统计

一次页面访问可以同时做：

```redis
INCR page:pv:article:1001:20260705
PFADD page:uv:article:1001:20260705 user:2001
```

Go 中使用 Pipeline：

```go
func RecordPageView(ctx context.Context, rdb *redis.Client, pageID string, userID int64, now time.Time) error {
    date := now.Format("20060102")
    pvKey := fmt.Sprintf("page:pv:%s:%s", pageID, date)
    uvKey := fmt.Sprintf("page:uv:%s:%s", pageID, date)

    pipe := rdb.Pipeline()
    pipe.Incr(ctx, pvKey)
    pipe.PFAdd(ctx, uvKey, fmt.Sprintf("user:%d", userID))
    pipe.Expire(ctx, pvKey, 90*24*time.Hour)
    pipe.Expire(ctx, uvKey, 90*24*time.Hour)
    _, err := pipe.Exec(ctx)
    return err
}
```

---

## 十一、匿名用户怎么统计

如果用户未登录，可以使用：

- 设备 ID。
- Cookie ID。
- 浏览器指纹。
- IP + User-Agent 的弱标识。

示例：

```text
visitor:device_abc123
```

但这些都不是完美标识。

匿名 UV 本来就有统计误差，业务要能接受。

---

## 十二、常见错误

### 1. 用 HyperLogLog 做发奖人数

它有误差，不适合强精确业务。

### 2. 需要用户列表却用了 HyperLogLog

HyperLogLog 不能列出成员。

### 3. 不设置 TTL

每天、每页面的 UV key 会长期堆积。

### 4. 把 PV 和 UV 混在一起

PV 是次数，UV 是去重人数，模型不同。

### 5. 小数据量也盲目使用

如果只有几十个用户且要精确，Set 更简单。

---

## 十三、本节练习

请完成下面练习：

1. 解释 PV 和 UV 的区别。
2. 用 Set 统计 UV 会有什么问题？
3. 写出 `PFADD` 统计文章 UV 的命令。
4. 写出 `PFCOUNT` 查询 UV 的命令。
5. 用 Go 实现 `AddPageUV`。
6. 用 `PFMERGE` 合并一周 UV。
7. 说明 HyperLogLog 的误差限制。
8. 判断发奖人数统计是否适合 HyperLogLog。

---

## 十四、本节小结

这一节你学习了 HyperLogLog。

你需要记住：

- HyperLogLog 适合去重计数。
- 它占用内存很小，但结果有少量误差。
- `PFADD` 添加元素，`PFCOUNT` 统计数量，`PFMERGE` 合并统计。
- 它不能列出成员。
- 页面 UV、大盘趋势适合用它，计费和发奖不适合。

下一节我们学习 GEO，用 Redis 做附近门店查询。

