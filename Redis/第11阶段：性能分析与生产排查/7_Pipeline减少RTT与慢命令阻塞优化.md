# 7. Pipeline 减少 RTT 与慢命令阻塞优化

Redis 本身执行命令很快，但网络往返 RTT 会放大大量小命令的耗时。

Pipeline 可以把多条命令一次发送给 Redis，减少 RTT。

学完这一节后，你应该能够：

- 理解 RTT 对 Redis 性能的影响。
- 使用 Pipeline 批量执行命令。
- 区分 Pipeline 和事务。
- 知道 Pipeline 的大小控制。
- 优化慢命令和阻塞命令。

---

## 一、RTT 是什么

RTT：

```text
Round Trip Time，一次请求从客户端到服务端再返回的往返时间。
```

假设 RTT 是 1ms。

执行 1000 条命令：

```text
不用 Pipeline：约 1000 次 RTT
使用 Pipeline：可以少量 RTT 完成
```

这就是 Pipeline 的价值。

---

## 二、没有 Pipeline 的循环

```go
for _, id := range ids {
    key := fmt.Sprintf("article:detail:%d", id)
    val, err := rdb.Get(ctx, key).Result()
    if err != nil {
        // handle
    }
    _ = val
}
```

如果有 100 个 id，就是 100 次网络往返。

接口很容易慢在网络 RTT 上。

---

## 三、使用 Pipeline

```go
pipe := rdb.Pipeline()
cmds := make([]*redis.StringCmd, 0, len(ids))

for _, id := range ids {
    key := fmt.Sprintf("article:detail:%d", id)
    cmds = append(cmds, pipe.Get(ctx, key))
}

_, err := pipe.Exec(ctx)
if err != nil && !errors.Is(err, redis.Nil) {
    return err
}

for _, cmd := range cmds {
    val, err := cmd.Result()
    if errors.Is(err, redis.Nil) {
        continue
    }
    if err != nil {
        return err
    }
    _ = val
}
```

这样可以显著减少 RTT。

---

## 四、Pipeline 不是事务

Pipeline 只是批量发送命令。

它不保证原子性。

如果需要事务语义，可以使用：

```go
pipe := rdb.TxPipeline()
```

或 Lua。

但很多批量读写只是为了减少网络往返，普通 Pipeline 就够了。

---

## 五、Pipeline 大小控制

Pipeline 不能无限大。

太大可能导致：

- 客户端内存增长。
- Redis 一次处理过多命令。
- 响应包太大。
- 阻塞其他请求。
- 超时。

建议分批：

```go
const batchSize = 100
```

每批执行 100 或 500 条，具体看命令和 value 大小。

---

## 六、MGET 和 Pipeline 怎么选

如果只是批量 `GET`：

```redis
MGET key1 key2 key3
```

通常更简单。

但在 Cluster 中，`MGET` 要求 key 同槽。

Pipeline 可以让 ClusterClient 按 slot 分发命令。

选择：

- 单实例同类批量 GET：`MGET`。
- 多类型命令组合：Pipeline。
- Cluster 跨槽批量：ClusterClient Pipeline。

---

## 七、慢命令优化原则

慢命令常见特点：

```text
一次处理太多数据。
```

优化原则：

- 控制返回数量。
- 分页读取。
- 使用 SCAN 系列。
- 拆分 Big Key。
- 避免复杂 Lua。
- 避免高峰期大删除。
- 使用 `UNLINK`。

不要指望单纯增加连接数解决慢命令。

---

## 八、阻塞命令

阻塞命令：

```redis
BLPOP queue 0
XREAD BLOCK 0 STREAMS stream $
```

它们会阻塞客户端连接。

建议：

- 使用单独 Redis client。
- 单独连接池。
- 设置合理 BLOCK 时间。
- worker 和普通请求隔离。

否则阻塞消费可能占满普通业务连接池。

---

## 九、批量删除优化

不要：

```redis
KEYS cache:*
DEL key1 key2 key3 ...
```

推荐：

```text
SCAN 分批找 key
UNLINK 分批删除
```

伪代码：

```text
cursor = 0
do:
  cursor, keys = SCAN cursor MATCH cache:* COUNT 500
  UNLINK keys...
while cursor != 0
```

生产执行前要评估影响。

---

## 十、Go 中批量删除

```go
func UnlinkByPattern(ctx context.Context, rdb *redis.Client, pattern string) error {
    var cursor uint64
    for {
        keys, next, err := rdb.Scan(ctx, cursor, pattern, 500).Result()
        if err != nil {
            return err
        }
        cursor = next

        if len(keys) > 0 {
            if err := rdb.Unlink(ctx, keys...).Err(); err != nil {
                return err
            }
        }

        if cursor == 0 {
            return nil
        }
    }
}
```

注意：

```text
这仍然是危险操作，生产要限速、低峰执行、可中断。
```

---

## 十一、对比实验

实验：

```text
1. 循环执行 1000 次 SET。
2. 使用 Pipeline 执行 1000 次 SET。
3. 对比耗时。
```

你会看到 Pipeline 通常明显更快。

但如果每个 value 很大，Pipeline 太大也会带来问题。

所以要分批。

---

## 十二、常见错误

### 1. 循环里逐条 GET

大量 RTT 导致接口变慢。

### 2. Pipeline 当事务

Pipeline 不保证原子性。

### 3. Pipeline 一次塞太多命令

可能导致大响应和超时。

### 4. 阻塞命令占用普通连接池

普通请求会等待连接。

### 5. 批量删除用 KEYS

生产应使用 SCAN + UNLINK。

---

## 十三、本节练习

请完成下面练习：

1. 写一个循环 1000 次 `SET` 的测试。
2. 写一个 Pipeline 1000 次 `SET` 的测试。
3. 对比耗时。
4. 解释 Pipeline 为什么能减少 RTT。
5. 解释 Pipeline 和事务的区别。
6. 实现 SCAN + UNLINK 批量删除。
7. 为阻塞 Stream worker 单独创建 Redis client。

---

## 十四、本节小结

这一节你学习了 Pipeline 和慢命令优化。

你需要记住：

- Pipeline 通过批量发送命令减少 RTT。
- Pipeline 不等于事务。
- Pipeline 要控制批次大小。
- 慢命令要从数据量、Big Key、Lua 和删除方式上治理。
- 阻塞命令要和普通请求隔离连接池。

下一节我们做第 11 阶段综合排查实践。

