# 8. 实践：开启 AOF、配置 maxmemory、观察淘汰、治理 Big Key

这一节把第 9 阶段串起来做一次实践。

你会完成：

- 开启 AOF。
- 写入数据后重启 Redis，观察恢复。
- 对比 RDB 和 AOF。
- 设置 `maxmemory` 和淘汰策略。
- 构造 Big Key。
- 使用 `UNLINK` 删除 Big Key。

学完这一节后，你应该能够把持久化、过期和内存管理放进真实 Redis 运维思维中。

---

## 一、准备 Docker Compose

可以创建一个测试目录：

```text
redis-persistence-lab/
  docker-compose.yml
  redis.conf
  data/
```

`docker-compose.yml`：

```yaml
services:
  redis:
    image: redis:7
    container_name: redis-lab
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    ports:
      - "6379:6379"
    volumes:
      - ./redis.conf:/usr/local/etc/redis/redis.conf
      - ./data:/data
```

---

## 二、基础 redis.conf

```conf
dir /data

save 900 1
save 300 10
save 60 10000

appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes

maxmemory 64mb
maxmemory-policy allkeys-lru
```

说明：

- RDB 和 AOF 都开启。
- AOF 使用 `everysec`。
- 开启混合持久化。
- 限制最大内存 64MB。
- 使用 `allkeys-lru` 淘汰。

这是实验配置，不一定适合生产。

---

## 三、启动 Redis

```bash
docker compose up -d
```

连接：

```bash
docker exec -it redis-lab redis-cli
```

检查配置：

```redis
CONFIG GET appendonly
CONFIG GET appendfsync
CONFIG GET aof-use-rdb-preamble
CONFIG GET maxmemory
CONFIG GET maxmemory-policy
```

---

## 四、写入数据并观察恢复

写入：

```redis
SET article:detail:1001 "redis persistence"
INCR counter:article:view:1001
```

确认：

```redis
GET article:detail:1001
GET counter:article:view:1001
```

重启容器：

```bash
docker restart redis-lab
```

再次查询：

```redis
GET article:detail:1001
GET counter:article:view:1001
```

如果 AOF 正常，数据应该能恢复。

---

## 五、查看持久化状态

```redis
INFO persistence
```

关注：

```text
rdb_last_bgsave_status
aof_enabled
aof_last_bgrewrite_status
aof_current_size
aof_base_size
```

手动触发：

```redis
BGSAVE
BGREWRITEAOF
```

再查看：

```redis
INFO persistence
```

---

## 六、查看持久化文件

退出 redis-cli 后：

```bash
ls -lh data
```

可能看到：

```text
dump.rdb
appendonlydir/
```

Redis 版本不同，AOF 文件组织方式可能不同。

重点观察：

- RDB 文件是否生成。
- AOF 相关文件是否生成。
- 文件大小是否变化。

---

## 七、测试 maxmemory 和淘汰

确认策略：

```redis
CONFIG GET maxmemory-policy
```

写入一些数据：

```redis
SET k:1 value
SET k:2 value
SET k:3 value
```

用脚本或批量命令写入大量 key，直到触发淘汰。

观察：

```redis
INFO stats
```

关注：

```text
evicted_keys
```

如果这个值增加，说明 Redis 已经开始淘汰 key。

---

## 八、测试 noeviction

切换策略：

```redis
CONFIG SET maxmemory-policy noeviction
```

继续写入大量数据。

内存达到上限后，写命令可能报错：

```text
OOM command not allowed when used memory > 'maxmemory'
```

这说明 `noeviction` 不淘汰 key，而是拒绝写入。

测试完可以切回：

```redis
CONFIG SET maxmemory-policy allkeys-lru
```

---

## 九、构造 Big Key

构造一个较大的 Set：

```redis
SADD big:set 1
SADD big:set 2
SADD big:set 3
```

真实测试可以用脚本批量写入很多 member。

查看元素数量：

```redis
SCARD big:set
```

查看内存：

```redis
MEMORY USAGE big:set
```

如果只是学习，不要在生产 Redis 上构造大 key。

---

## 十、删除 Big Key

普通删除：

```redis
DEL big:set
```

更推荐：

```redis
UNLINK big:set
```

`UNLINK` 会异步释放内存，降低阻塞风险。

生产治理 Big Key 时，优先考虑：

- 拆分。
- 分批迁移。
- 使用 `UNLINK`。
- 低峰期操作。
- 提前备份。

---

## 十一、检查 key 大小

常用命令：

```redis
MEMORY USAGE key
STRLEN string:key
HLEN hash:key
LLEN list:key
SCARD set:key
ZCARD zset:key
```

扫描工具：

```bash
redis-cli --bigkeys
```

生产运行扫描要谨慎，避免影响 Redis。

---

## 十二、实验记录模板

建议记录：

```text
Redis 版本：
appendonly：
appendfsync：
aof-use-rdb-preamble：
maxmemory：
maxmemory-policy：
BGSAVE 耗时：
BGREWRITEAOF 耗时：
evicted_keys 变化：
Big Key 类型：
Big Key MEMORY USAGE：
删除方式：DEL / UNLINK
```

做过记录，才算真正理解配置影响。

---

## 十三、生产配置检查清单

上线前至少确认：

- 是否需要 RDB。
- 是否需要 AOF。
- `appendfsync` 策略是否符合丢数据要求。
- 是否开启混合持久化。
- 是否配置 `maxmemory`。
- 淘汰策略是否符合业务。
- 缓存 key 是否都有 TTL。
- 是否存在 Big Key。
- 是否监控 `expired_keys`、`evicted_keys`。
- 是否监控持久化失败。
- 是否有备份和恢复演练。

这张清单比背参数更重要。

---

## 十四、常见错误

### 1. 实验配置直接用于生产

生产参数要根据内存、磁盘、业务和 SLA 调整。

### 2. 重启恢复成功一次就以为万无一失

还要做备份恢复演练。

### 3. 只看 maxmemory，不看淘汰策略

内存满后的行为由策略决定。

### 4. 在生产随意构造 Big Key

可能造成阻塞和内存压力。

### 5. 删除大 key 直接 DEL

优先考虑 `UNLINK` 和低峰期操作。

---

## 十五、本节练习

请完成下面练习：

1. 使用 Docker Compose 启动 Redis。
2. 开启 AOF 和混合持久化。
3. 写入数据后重启 Redis，观察恢复。
4. 执行 `BGSAVE` 和 `BGREWRITEAOF`。
5. 查看 RDB 和 AOF 文件。
6. 设置较小 `maxmemory`。
7. 对比 `noeviction` 和 `allkeys-lru`。
8. 构造一个测试 Big Key。
9. 使用 `MEMORY USAGE` 查看大小。
10. 使用 `UNLINK` 删除 Big Key。

---

## 十六、本节小结

这一节完成了第 9 阶段的实践闭环。

你需要记住：

- AOF 可以提高 Redis 重启后的数据恢复能力。
- `appendfsync everysec` 是常见平衡选择。
- 混合持久化可以改善 AOF 恢复速度。
- `maxmemory` 和 `maxmemory-policy` 决定内存满时的行为。
- `evicted_keys` 能反映淘汰是否发生。
- Big Key 要通过拆分、分片、控制大小和 `UNLINK` 治理。

学完第 9 阶段，你已经理解 Redis 数据如何保存、过期 key 如何清理、内存不足时如何淘汰，以及 Big Key 为什么会成为生产风险。下一阶段会进入主从复制、哨兵与集群。

