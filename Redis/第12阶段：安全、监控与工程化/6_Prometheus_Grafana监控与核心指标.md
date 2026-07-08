# 6. Prometheus / Grafana 监控与核心指标

没有监控的 Redis，就像没有仪表盘的车。

你可能能开一段路，但出了问题很难知道发生了什么。

学完这一节后，你应该能够：

- 理解 Redis 监控的目标。
- 知道 Redis Exporter 的作用。
- 设计 Prometheus/Grafana 监控指标。
- 为常见异常设置告警。
- 建立 Redis 观测面板。

---

## 一、监控目标

Redis 监控要回答：

```text
Redis 是否还活着？
请求量是否异常？
延迟是否升高？
内存是否接近上限？
是否发生淘汰？
命中率是否下降？
连接是否异常？
持久化是否失败？
复制是否延迟？
慢命令是否增加？
```

监控不是为了好看。

它要帮助你提前发现故障。

---

## 二、Redis Exporter

Prometheus 通常通过 Redis Exporter 采集 Redis 指标。

结构：

```text
Redis -> redis_exporter -> Prometheus -> Grafana
```

Exporter 会连接 Redis，读取 `INFO` 等指标，并暴露 HTTP metrics。

常见指标会映射成 Prometheus 格式。

---

## 三、Docker Compose 示例

示意：

```yaml
services:
  redis:
    image: redis:7

  redis-exporter:
    image: oliver006/redis_exporter
    environment:
      REDIS_ADDR: redis://redis:6379
      REDIS_USER: app
      REDIS_PASSWORD: app-password
    ports:
      - "9121:9121"
    depends_on:
      - redis
```

生产环境镜像版本、认证和网络要按公司规范固定。

---

## 四、核心指标：存活与连接

关注：

- Redis 是否可达。
- connected clients。
- blocked clients。
- rejected connections。

告警示例：

```text
Redis down 超过 1 分钟。
connected_clients 突然超过平时 3 倍。
blocked_clients > 0 持续 5 分钟。
rejected_connections 增长。
```

连接异常通常和应用连接池、重连风暴、阻塞命令有关。

---

## 五、核心指标：QPS 和命中率

关注：

- instantaneous ops per sec。
- total commands processed。
- keyspace hits。
- keyspace misses。

命中率：

```text
hits / (hits + misses)
```

告警：

```text
命中率低于历史基线。
misses 突然升高。
QPS 突然升高或突降。
```

命中率下降通常会传导到数据库。

---

## 六、核心指标：内存

关注：

- used memory。
- maxmemory。
- memory usage ratio。
- mem fragmentation ratio。
- evicted keys。

告警：

```text
内存使用率 > 80%。
evicted_keys 持续增长。
mem_fragmentation_ratio 长期过高。
```

内存告警要结合业务类型。

纯缓存淘汰可能正常。

重要状态被淘汰就很危险。

---

## 七、核心指标：慢命令

关注：

- slowlog length。
- 慢命令数量增长。
- commandstats 中异常命令耗时。

告警：

```text
慢命令数量在 5 分钟内持续增长。
```

慢命令出现后，要结合：

```redis
SLOWLOG GET
```

定位具体命令和 key。

---

## 八、核心指标：持久化

关注：

- RDB last save status。
- AOF enabled。
- AOF rewrite status。
- AOF current size。
- COW size。

告警：

```text
RDB 或 AOF 最近一次失败。
AOF 文件增长异常。
AOF rewrite 长时间进行。
```

持久化失败不一定立即影响接口，但会影响故障恢复能力。

---

## 九、核心指标：复制

主从/Sentinel/Cluster 环境关注：

- master link status。
- connected replicas。
- replication offset lag。
- role。
- master changes。

告警：

```text
replica 断开。
复制延迟超过阈值。
主从切换发生。
```

读写分离场景下，复制延迟会影响读一致性。

---

## 十、Grafana 面板建议

建议面板：

```text
1. 实例存活。
2. QPS。
3. 命中率。
4. 内存使用率。
5. evicted_keys / expired_keys。
6. connected_clients / blocked_clients。
7. 慢命令。
8. 网络输入输出。
9. 持久化状态。
10. 复制状态和延迟。
```

面板要按服务和实例区分。

多个 Redis 用途不同，阈值也不同。

---

## 十一、告警分级

示例：

| 级别 | 场景 |
| --- | --- |
| P1 | Redis 不可用、主从全部失败、数据大量丢失 |
| P2 | 内存接近上限、淘汰异常、复制断开 |
| P3 | 命中率下降、慢命令增加、连接数异常 |

告警要可行动。

如果告警太多没人处理，就等于没有告警。

---

## 十二、常见错误

### 1. 只监控 Redis 存活

PING 正常不代表性能正常。

### 2. 没有命中率监控

缓存问题会先表现为数据库压力。

### 3. 所有 Redis 用同一告警阈值

缓存、队列、锁、排行榜的指标基线不同。

### 4. 告警无人负责

没有负责人和流程，告警没有意义。

### 5. 没有客户端指标

服务端正常，客户端连接池也可能耗尽。

---

## 十三、本节练习

请完成下面练习：

1. 列出 Redis 监控的 10 个核心指标。
2. 设计一个 Redis Grafana 面板。
3. 为内存使用率设置告警。
4. 为命中率下降设置告警。
5. 为复制断开设置告警。
6. 思考缓存 Redis 和 Stream Redis 的告警阈值是否应该一样。
7. 给 go-redis PoolStats 也设计监控指标。

---

## 十四、本节小结

这一节你学习了 Redis 监控。

你需要记住：

- Redis 监控要覆盖存活、QPS、命中率、内存、连接、慢命令、持久化和复制。
- Prometheus 通常通过 Redis Exporter 采集指标。
- Grafana 面板要能支持排查，而不是只展示漂亮曲线。
- 告警要分级、有负责人、可行动。
- 客户端指标和 Redis 服务端指标同样重要。

下一节我们学习备份、恢复和演练流程。

