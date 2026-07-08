# 5. Sentinel：主观下线、客观下线与故障转移

主从复制提供副本，但不会自动切主。

Sentinel 负责监控 Redis 主从节点，并在 master 故障时自动完成故障转移。

学完这一节后，你应该能够：

- 理解 Sentinel 的作用。
- 区分主观下线和客观下线。
- 理解故障转移基本流程。
- 搭建简单 Sentinel 环境。
- 知道 Sentinel 也不能保证零数据丢失。

---

## 一、Sentinel 是什么

Sentinel 是 Redis 的高可用组件。

它负责：

- 监控 master。
- 监控 replica。
- 判断故障。
- 选举 Sentinel leader。
- 选择新 master。
- 让其他 replica 复制新 master。
- 通知客户端 master 地址变化。

Sentinel 通常至少部署 3 个。

---

## 二、为什么要多个 Sentinel

如果只有一个 Sentinel：

```text
Sentinel 自己挂了，就没人判断故障。
Sentinel 网络抖动，可能误判 master。
```

多个 Sentinel 可以投票。

只有达到法定数量，才认为 master 客观下线。

这能降低误判风险。

---

## 三、主观下线 SDOWN

主观下线：

```text
某一个 Sentinel 认为 master 不可达。
```

例如 Sentinel 连续一段时间 ping master 不通。

它会把 master 标记为：

```text
SDOWN
```

Subjectively Down。

主观下线只是单个 Sentinel 的判断。

---

## 四、客观下线 ODOWN

客观下线：

```text
多个 Sentinel 达成一致，认为 master 不可达。
```

它会把 master 标记为：

```text
ODOWN
```

Objectively Down。

只有客观下线后，才会触发故障转移。

---

## 五、故障转移基本流程

简化流程：

```text
1. Sentinel 发现 master 主观下线。
2. 多个 Sentinel 确认 master 客观下线。
3. Sentinel 之间选举一个 leader。
4. leader 从 replica 中选择一个提升为 master。
5. 其他 replica 改为复制新 master。
6. 旧 master 如果恢复，会变成新 master 的 replica。
7. Sentinel 通知客户端新 master 地址。
```

故障转移期间，客户端可能短暂写失败。

---

## 六、选择新 master 的考虑

Sentinel 会综合考虑：

- replica 是否在线。
- replica 优先级。
- 复制 offset。
- run id。

直觉上：

```text
越新、越健康、优先级越高的 replica 越适合被提升。
```

如果 replica 落后很多，被提升后可能丢更多数据。

---

## 七、Sentinel 配置示例

`sentinel.conf`：

```conf
port 26379
sentinel monitor mymaster redis-master 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

含义：

- `mymaster` 是 master 名称。
- `redis-master 6379` 是 master 地址。
- `2` 是 quorum，至少 2 个 Sentinel 同意才客观下线。
- 5 秒不可达后主观下线。

---

## 八、Docker Compose 简化环境

结构：

```text
redis-master
redis-replica
sentinel-1
sentinel-2
sentinel-3
```

学习时可以先手写配置。

生产中建议使用成熟部署方案。

Sentinel 配置会自动重写，配置文件需要可写。

这点在 Docker 挂载配置时要注意。

---

## 九、查看 Sentinel 状态

连接 Sentinel：

```bash
redis-cli -p 26379
```

查看 master：

```redis
SENTINEL masters
```

查看 replicas：

```redis
SENTINEL replicas mymaster
```

获取当前 master 地址：

```redis
SENTINEL get-master-addr-by-name mymaster
```

客户端会用类似能力发现 master。

---

## 十、手动模拟故障

停止 master：

```bash
docker stop redis-master
```

观察 Sentinel 日志。

查看当前 master：

```redis
SENTINEL get-master-addr-by-name mymaster
```

如果故障转移成功，返回的应该是原 replica 的地址。

---

## 十一、Sentinel 的数据丢失窗口

Sentinel 不能改变 Redis 异步复制本质。

如果 master 写入后还没复制给 replica 就宕机：

```text
新 master 上没有这条数据。
```

所以 Sentinel 提供高可用，不提供强一致。

核心数据仍然要由数据库保证。

---

## 十二、常见错误

### 1. 只部署一个 Sentinel

容易单点和误判。

### 2. quorum 设置不合理

太小容易误判，太大可能无法切换。

### 3. 客户端仍然连固定 master 地址

故障转移后客户端找不到新 master。

### 4. 以为 Sentinel 不会丢数据

异步复制下仍可能丢未复制写入。

### 5. Sentinel 配置文件不可写

Sentinel 需要重写配置保存拓扑变化。

---

## 十三、本节练习

请完成下面练习：

1. 用自己的话解释 Sentinel。
2. 区分主观下线和客观下线。
3. 写出 `sentinel monitor` 配置。
4. 查看 `SENTINEL masters`。
5. 停止 master，观察故障转移。
6. 查询新 master 地址。
7. 思考 Sentinel 为什么仍可能丢数据。

---

## 十四、本节小结

这一节你学习了 Sentinel。

你需要记住：

- Sentinel 负责 Redis 主从架构的监控和自动故障转移。
- 主观下线是单个 Sentinel 的判断。
- 客观下线是多个 Sentinel 达成一致。
- 故障转移会选择一个 replica 提升为新 master。
- Sentinel 提供高可用，但不能保证零数据丢失。

下一节我们学习 Go 客户端如何连接 Sentinel。

