# 8. 实践：Redis 慢、抖动、内存异常、热点问题排查清单

这一节把第 11 阶段串成一份实战清单。

目标是当生产里出现“Redis 慢了”“接口超时了”“内存涨了”“数据库被打爆了”时，你知道按什么顺序查。

学完这一节后，你应该能够：

- 使用一套固定流程排查 Redis 问题。
- 根据现象选择命令和指标。
- 区分服务端、客户端、网络和业务问题。
- 制定止血方案。
- 输出复盘和长期优化项。

---

## 一、场景 1：接口 Redis 超时

现象：

```text
接口 P99 从 80ms 涨到 2s。
日志里出现 redis timeout。
```

排查：

```text
1. 看应用日志：命令、key、错误类型。
2. 看 go-redis PoolStats：Timeouts 是否增长。
3. 看 SLOWLOG：是否有慢命令。
4. 看 INFO clients：连接数是否异常。
5. 看 INFO stats：QPS 是否突增。
6. 看网络 RTT。
7. 看最近发布和活动流量。
```

可能原因：

- 连接池耗尽。
- 网络抖动。
- Big Key。
- Redis CPU 高。
- 慢 Lua。
- 热点 key。

止血：

- 降级非核心 Redis 读写。
- 限流异常接口。
- 临时扩容应用或 Redis。
- 关闭新上线功能。

---

## 二、场景 2：SLOWLOG 出现大集合命令

现象：

```redis
SLOWLOG GET 10
```

看到：

```text
SMEMBERS big:set
HGETALL big:hash
LRANGE big:list 0 -1
```

处理：

```text
1. 定位调用接口。
2. 查看 MEMORY USAGE。
3. 查看集合元素数量。
4. 改成分页、SCAN 或拆 key。
5. 加返回数量限制。
6. 增加慢命令监控。
```

不要只是把 slowlog 清掉。

---

## 三、场景 3：内存持续上涨

排查：

```redis
INFO memory
INFO stats
INFO keyspace
MEMORY STATS
```

继续：

```text
1. used_memory 是否持续上涨。
2. evicted_keys 是否增长。
3. keyspace 中 keys 是否增长。
4. expires 占比是否异常。
5. 抽样 SCAN 检查 TTL。
6. MEMORY USAGE 找大 key。
7. 查看最近是否新增缓存类型。
```

可能原因：

- 缓存没 TTL。
- 新增大 value。
- 排行榜不清理。
- Stream 不修剪。
- Big Key。
- 活动流量产生大量临时 key。

---

## 四、场景 4：命中率突然下降

计算：

```text
hit_rate = keyspace_hits / (keyspace_hits + keyspace_misses)
```

可能原因：

- 大量 key 同时过期。
- key 命名变更。
- 缓存写入失败。
- Redis 淘汰。
- 缓存穿透。
- 新接口未接缓存。

排查：

```text
1. 看 keyspace_misses 是否突增。
2. 看 evicted_keys 是否增长。
3. 看 expired_keys 是否突增。
4. 看数据库 QPS 是否上涨。
5. 看发布记录。
6. 抽样检查 key 是否存在。
```

止血：

- 限流回源。
- 热点 key 预热。
- 延长 TTL。
- 启用空值缓存。
- 修复 key 命名。

---

## 五、场景 5：Hot Key 打爆 Redis

现象：

- 单节点 CPU 高。
- 某接口 QPS 极高。
- 应用日志集中访问某个 key。
- Cluster 中某个 slot 节点压力特别大。

处理：

```text
1. 应用侧统计 key 访问频率。
2. 确认热点 key 和接口。
3. 判断 value 是否也很大。
4. 加本地缓存。
5. 使用逻辑过期。
6. 必要时使用热点副本。
7. 限流或降级。
```

如果 key 又大又热，先拆大。

---

## 六、场景 6：连接数暴涨

排查：

```redis
INFO clients
CLIENT LIST
```

应用侧：

```go
rdb.PoolStats()
```

可能原因：

- 应用发布后实例数增加。
- 每次请求创建 Redis client。
- 连接池过大。
- 重连风暴。
- 阻塞命令占连接。

处理：

- 复用 Redis client。
- 调整 PoolSize。
- 设置 PoolTimeout。
- 阻塞消费单独 client。
- 给客户端设置 ClientName。

---

## 七、场景 7：Pipeline 优化批量读取

问题：

```text
文章列表页要读取 100 篇文章缓存。
循环 GET 导致 100 次 RTT。
```

优化：

- 单实例可以用 `MGET`。
- Cluster 跨槽可以用 Pipeline。
- 控制批次大小。
- 未命中批量回源数据库。

注意：

```text
Pipeline 降低 RTT，不解决 Big Key 和慢命令。
```

---

## 八、统一排查命令清单

```redis
INFO stats
INFO memory
INFO clients
INFO persistence
INFO replication
INFO commandstats
SLOWLOG GET 20
CLIENT LIST
MEMORY STATS
MEMORY USAGE key
TTL key
SCAN 0 MATCH pattern COUNT 100
```

生产慎用：

```redis
MONITOR
KEYS *
```

---

## 九、Go 服务指标清单

建议应用侧监控：

- Redis 操作耗时 P50/P95/P99。
- Redis 错误数。
- Redis timeout 数。
- 连接池 `Timeouts`。
- 连接池 `TotalConns`。
- 连接池 `IdleConns`。
- 缓存命中率。
- 回源数据库次数。
- 单 key 访问频率采样。
- 降级次数。

服务端指标和客户端指标要一起看。

---

## 十、止血策略清单

常见止血：

- 限流异常接口。
- 降级非核心 Redis 功能。
- 暂停批任务。
- 热点 key 加本地缓存。
- 临时预热缓存。
- 临时扩容。
- 调整连接池。
- 使用 `UNLINK` 删除异常 Big Key。
- 回滚发布。

止血动作要记录时间点。

方便后续复盘。

---

## 十一、复盘模板

```text
故障时间：
影响接口：
影响用户：
主要现象：
第一条告警：
关键指标变化：
根因：
止血动作：
恢复时间：
是否丢数据：
后续优化：
负责人：
截止时间：
```

没有复盘，问题很容易重复发生。

---

## 十二、常见错误

### 1. 没有客户端指标

只看 Redis 服务端，无法发现连接池耗尽。

### 2. 故障时直接 KEYS

可能让 Redis 更慢。

### 3. 只扩容不修业务

Big Key 和 Hot Key 迟早再出问题。

### 4. 没有降级开关

故障时只能硬扛。

### 5. 没有复盘行动项

同类问题会重复出现。

---

## 十三、本节练习

请完成下面练习：

1. 整理一份 Redis 超时排查流程。
2. 整理一份内存上涨排查流程。
3. 整理一份 Hot Key 治理方案。
4. 写一个 Pipeline 批量读取示例。
5. 给 go-redis PoolStats 接入日志或指标。
6. 设计一个 Redis 故障降级开关。
7. 写一份 Redis 故障复盘模板。

---

## 十四、本节小结

这一节完成了第 11 阶段的实践闭环。

你需要记住：

- Redis 排查要同时看服务端、客户端、网络和业务访问模式。
- `INFO`、`SLOWLOG`、`CLIENT LIST`、`MEMORY` 是基础工具。
- Big Key 和 Hot Key 是生产高频问题。
- Pipeline 能减少 RTT，但不能解决所有慢问题。
- 故障处理要先止血，再定位，最后复盘优化。

学完第 11 阶段，你已经具备 Redis 性能分析和生产排查的基本能力。下一阶段会进入安全、监控与工程化。

