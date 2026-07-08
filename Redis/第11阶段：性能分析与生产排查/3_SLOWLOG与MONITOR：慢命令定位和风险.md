# 3. SLOWLOG 与 MONITOR：慢命令定位和风险

Redis 是单线程执行命令的主循环模型。

一个慢命令可能阻塞后面的命令，导致接口抖动。`SLOWLOG` 用来记录慢命令，`MONITOR` 可以实时观察命令，但生产中要非常谨慎。

学完这一节后，你应该能够：

- 使用 `SLOWLOG GET` 查看慢命令。
- 配置慢命令阈值。
- 理解慢命令常见来源。
- 知道 `MONITOR` 的风险。
- 用慢命令结果指导优化。

---

## 一、什么是慢命令

慢命令不是指网络耗时慢。

Redis `SLOWLOG` 记录的是：

```text
命令在 Redis 服务端执行耗时超过阈值。
```

不包含：

- 网络传输时间。
- 客户端等待连接时间。
- 客户端反序列化时间。

所以应用里看到 200ms，不代表 SLOWLOG 一定有 200ms。

---

## 二、查看慢命令

```redis
SLOWLOG GET
```

查看最近 10 条：

```redis
SLOWLOG GET 10
```

返回信息包含：

- 慢日志 ID。
- 时间戳。
- 执行耗时，单位微秒。
- 命令和参数。
- 客户端地址。

---

## 三、慢命令阈值

查看配置：

```redis
CONFIG GET slowlog-log-slower-than
```

单位是微秒。

例如：

```text
10000
```

表示超过 10ms 的命令记录到 slowlog。

设置：

```redis
CONFIG SET slowlog-log-slower-than 10000
```

生产中不要随便设得太低，否则日志噪音很大。

---

## 四、慢日志长度

查看：

```redis
CONFIG GET slowlog-max-len
```

设置：

```redis
CONFIG SET slowlog-max-len 128
```

`SLOWLOG` 是内存中的环形日志。

超过长度后旧记录会被覆盖。

监控系统应该定期采集，而不是故障后才想起来查。

---

## 五、常见慢命令来源

常见原因：

- `KEYS *`。
- 大范围 `HGETALL`。
- 大集合 `SMEMBERS`。
- 大列表 `LRANGE 0 -1`。
- 大 ZSet 范围查询。
- 慢 Lua 脚本。
- 删除 Big Key。
- 聚合类命令处理元素过多。

这些问题本质通常是：

```text
一次命令处理太多数据。
```

---

## 六、Big Key 导致慢命令

例如：

```redis
HGETALL big:hash
SMEMBERS big:set
LRANGE big:list 0 -1
```

如果 key 中有几十万元素，命令会很慢。

优化：

- 分页读取。
- 拆分 key。
- 限制返回范围。
- 使用 `HSCAN`、`SSCAN`、`ZSCAN`。
- 避免一次取全量。

---

## 七、慢 Lua

Lua 脚本会在 Redis 主线程执行。

如果脚本里：

- 大循环。
- 扫描大量 key。
- 处理大集合。
- 写复杂业务逻辑。

会阻塞其他命令。

Lua 应该短小，只做关键原子逻辑。

---

## 八、删除 Big Key

慢删除：

```redis
DEL big:set
```

更推荐：

```redis
UNLINK big:set
```

`UNLINK` 将释放内存放到后台线程，降低主线程阻塞。

但它不是治理 Big Key 的根本方案。

根本还是拆分和控制大小。

---

## 九、MONITOR 是什么

```redis
MONITOR
```

会实时打印 Redis 收到的所有命令。

它看起来很方便：

```text
哪个客户端在发什么命令，一眼能看到。
```

但生产中非常危险。

---

## 十、MONITOR 的风险

`MONITOR` 会让 Redis 把所有命令都输出给监听客户端。

风险：

- 增加 Redis 负担。
- 高 QPS 下输出量巨大。
- 可能暴露敏感参数。
- 可能导致监控客户端或网络压力。
- 可能进一步放大故障。

生产环境除非非常短时间、低峰、明确需要，否则不要使用。

更推荐通过：

- 慢日志。
- 应用日志。
- 采样日志。
- 代理层日志。
- 监控指标。

定位问题。

---

## 十一、如何处理慢命令

处理路径：

```text
1. 找到慢命令。
2. 找到 key 和业务接口。
3. 判断是否 Big Key。
4. 判断是否一次返回太多数据。
5. 改成分页、SCAN、拆 key 或缓存局部结果。
6. 加监控和限制，防止再次发生。
```

不要只把 slowlog 阈值调大。

阈值调大只是让你看不见问题。

---

## 十二、Go 中记录 Redis 命令耗时

可以在业务封装层记录：

```go
start := time.Now()
val, err := rdb.Get(ctx, key).Result()
cost := time.Since(start)

if cost > 50*time.Millisecond {
    logger.Printf("redis slow op cmd=GET key=%s cost=%s err=%v", key, cost, err)
}
```

应用侧耗时包括：

- 等连接。
- 网络。
- Redis 执行。
- 响应读取。

它和 SLOWLOG 是互补关系。

---

## 十三、常见错误

### 1. 用 MONITOR 长时间排查

可能放大生产故障。

### 2. 看到应用慢但 SLOWLOG 没记录就认为 Redis 没问题

可能是网络、连接池或客户端处理慢。

### 3. 慢命令只删除不治理

同样的 Big Key 或命令还会再次出现。

### 4. Lua 写复杂业务

慢 Lua 会阻塞 Redis 主线程。

### 5. 用 `KEYS *` 排查生产问题

生产应使用 `SCAN`，不要全库阻塞扫描。

---

## 十四、本节练习

请完成下面练习：

1. 查看 `SLOWLOG GET 10`。
2. 查看 `slowlog-log-slower-than`。
3. 设置慢日志阈值为 10ms。
4. 构造一个大集合并观察慢命令风险。
5. 比较 `DEL` 和 `UNLINK` 的适用场景。
6. 解释为什么生产慎用 `MONITOR`。
7. 在 Go 中记录一次 Redis 操作耗时。

---

## 十五、本节小结

这一节你学习了 `SLOWLOG` 和 `MONITOR`。

你需要记住：

- `SLOWLOG` 记录 Redis 服务端执行慢的命令。
- 它不包含网络和客户端等待时间。
- 慢命令常来自 Big Key、大范围查询、慢 Lua 和阻塞删除。
- `MONITOR` 能实时看命令，但生产风险很高。
- 应用侧耗时日志和 Redis SLOWLOG 要结合分析。

下一节我们学习 `CLIENT LIST` 和连接池排查。

