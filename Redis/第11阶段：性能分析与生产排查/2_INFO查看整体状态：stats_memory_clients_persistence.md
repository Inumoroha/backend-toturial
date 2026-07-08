# 2. INFO 查看整体状态：stats、memory、clients、persistence

`INFO` 是 Redis 排查的第一入口。

它会返回 Redis 服务端大量运行状态，帮助你判断当前 Redis 是否健康。

学完这一节后，你应该能够：

- 使用 `INFO` 查看 Redis 状态。
- 理解常用 section。
- 关注连接、内存、命中率、淘汰、持久化、复制等指标。
- 从 `INFO` 中找到排查线索。
- 知道哪些指标适合监控。

---

## 一、INFO 基本用法

查看全部：

```redis
INFO
```

查看某个 section：

```redis
INFO stats
INFO memory
INFO clients
INFO persistence
INFO replication
INFO commandstats
```

生产排查时，不要被全部输出淹没。

按问题选择 section。

---

## 二、server 信息

```redis
INFO server
```

关注：

```text
redis_version
uptime_in_seconds
tcp_port
process_id
```

用途：

- 确认 Redis 版本。
- 确认实例是否刚重启。
- 对齐日志和监控。

如果 `uptime_in_seconds` 很小，说明 Redis 可能刚发生过重启。

---

## 三、clients 信息

```redis
INFO clients
```

关注：

```text
connected_clients
blocked_clients
tracking_clients
```

含义：

- `connected_clients`：当前连接数。
- `blocked_clients`：阻塞等待的客户端数量，例如阻塞队列命令。

连接数突然升高，可能是：

- 应用连接池配置错误。
- 应用实例数增加。
- 连接泄漏。
- 客户端重连风暴。

---

## 四、memory 信息

```redis
INFO memory
```

关注：

```text
used_memory_human
used_memory_peak_human
maxmemory_human
mem_fragmentation_ratio
allocator_frag_ratio
```

含义：

- 当前内存使用。
- 历史峰值。
- 最大内存限制。
- 内存碎片情况。

如果 `used_memory` 接近 `maxmemory`，要继续看淘汰策略和 `evicted_keys`。

---

## 五、stats 信息

```redis
INFO stats
```

关注：

```text
total_commands_processed
instantaneous_ops_per_sec
total_net_input_bytes
total_net_output_bytes
rejected_connections
expired_keys
evicted_keys
keyspace_hits
keyspace_misses
```

常用判断：

- `instantaneous_ops_per_sec` 看当前 QPS。
- `evicted_keys` 增长说明内存淘汰发生。
- `expired_keys` 增长说明过期删除发生。
- `keyspace_hits/misses` 可计算命中率。

---

## 六、命中率计算

命中率：

```text
hit_rate = keyspace_hits / (keyspace_hits + keyspace_misses)
```

如果命中率突然下降，可能是：

- 大量 key 过期。
- 缓存雪崩。
- 新接口没有写缓存。
- key 命名变更。
- Redis 淘汰严重。
- 缓存穿透。

命中率下降常常会导致数据库压力上升。

---

## 七、persistence 信息

```redis
INFO persistence
```

关注：

```text
rdb_bgsave_in_progress
rdb_last_bgsave_status
rdb_last_cow_size
aof_enabled
aof_rewrite_in_progress
aof_last_bgrewrite_status
aof_current_size
aof_last_cow_size
```

用途：

- 判断是否正在 RDB/AOF 重写。
- 判断持久化是否失败。
- 判断 fork/COW 是否带来内存压力。

如果 Redis 抖动刚好发生在 AOF 重写期间，要重点关注。

---

## 八、replication 信息

```redis
INFO replication
```

关注：

```text
role
connected_slaves
master_link_status
master_repl_offset
slave_repl_offset
repl_backlog_size
```

用途：

- 判断主从角色。
- 判断复制是否断开。
- 观察复制延迟。
- 判断 backlog 是否足够。

主从切换或读写分离问题，需要看这一段。

---

## 九、commandstats 信息

```redis
INFO commandstats
```

示例：

```text
cmdstat_get:calls=100000,usec=300000,usec_per_call=3.00
cmdstat_set:calls=50000,usec=250000,usec_per_call=5.00
```

用途：

- 看哪些命令调用最多。
- 看平均耗时。
- 发现异常命令。

但慢请求细节还要看 `SLOWLOG`。

---

## 十、cpu 信息

```redis
INFO cpu
```

关注：

```text
used_cpu_sys
used_cpu_user
used_cpu_sys_children
used_cpu_user_children
```

如果子进程 CPU 增长明显，可能和：

- RDB。
- AOF 重写。
- 复制全量同步。

有关。

---

## 十一、keyspace 信息

```redis
INFO keyspace
```

示例：

```text
db0:keys=100000,expires=80000,avg_ttl=3600000
```

可以看：

- 每个 db 的 key 数量。
- 有过期时间的 key 数量。
- 平均 TTL。

如果缓存 key 大量没有 TTL，要回到 key 设计检查。

---

## 十二、排查示例：接口超时

如果接口 Redis 超时，先看：

```redis
INFO stats
INFO clients
INFO memory
INFO commandstats
```

判断：

- QPS 是否突然升高。
- 连接数是否异常。
- 是否有淘汰。
- 命中率是否下降。
- 某些命令调用是否异常。

再结合：

```redis
SLOWLOG GET
CLIENT LIST
MEMORY STATS
```

继续深入。

---

## 十三、适合监控的指标

建议监控：

- QPS。
- 命中率。
- 连接数。
- blocked clients。
- used memory。
- maxmemory 使用率。
- evicted keys。
- expired keys。
- rejected connections。
- slowlog 长度。
- AOF/RDB 状态。
- 主从复制状态和延迟。

监控比故障时手工查更重要。

---

## 十四、常见错误

### 1. 只看 used_memory

还要看 maxmemory、碎片率、淘汰、Big Key。

### 2. 只看平均命令耗时

慢命令和 P99 更重要。

### 3. 忽略 keyspace_misses

缓存未命中可能压垮数据库。

### 4. 忽略 persistence 状态

AOF/RDB 失败或重写可能影响性能。

### 5. 不做趋势监控

单次 INFO 只能看到当前状态，趋势更有价值。

---

## 十五、本节练习

请完成下面练习：

1. 执行 `INFO stats`。
2. 执行 `INFO memory`。
3. 执行 `INFO clients`。
4. 计算 keyspace 命中率。
5. 查看 `evicted_keys` 是否增长。
6. 查看 `connected_clients`。
7. 查看 `aof_rewrite_in_progress`。
8. 整理一份 Redis 监控指标清单。

---

## 十六、本节小结

这一节你学习了 `INFO`。

你需要记住：

- `INFO` 是 Redis 整体状态排查入口。
- `stats` 看 QPS、命中、淘汰和过期。
- `memory` 看内存和碎片。
- `clients` 看连接。
- `persistence` 看 RDB/AOF 状态。
- `replication` 看主从复制状态。
- 单次指标不如趋势监控有价值。

下一节我们学习 `SLOWLOG` 和 `MONITOR`。

