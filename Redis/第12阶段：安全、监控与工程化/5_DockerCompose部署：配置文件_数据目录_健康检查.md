# 5. Docker Compose 部署：配置文件、数据目录、健康检查

学习阶段可以直接 `docker run redis`。

但更工程化的方式是使用 Docker Compose 管理配置、数据目录、密码、健康检查和网络。

学完这一节后，你应该能够：

- 使用 Docker Compose 启动 Redis。
- 挂载 `redis.conf`。
- 挂载数据目录。
- 配置密码和持久化。
- 添加健康检查。
- 避免把 Redis 意外暴露到公网。

---

## 一、推荐目录结构

```text
redis-deploy/
  docker-compose.yml
  redis.conf
  users.acl
  data/
  README.md
```

说明：

- `docker-compose.yml` 管理容器。
- `redis.conf` 管理 Redis 配置。
- `users.acl` 管理 ACL 用户。
- `data/` 保存 RDB/AOF 文件。
- `README.md` 记录运行说明。

---

## 二、redis.conf 示例

```conf
bind 0.0.0.0
protected-mode yes

port 6379
dir /data

aclfile /usr/local/etc/redis/users.acl

appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes

maxmemory 512mb
maxmemory-policy allkeys-lru

slowlog-log-slower-than 10000
slowlog-max-len 128
```

注意：

```text
bind 0.0.0.0 只表示容器内监听所有地址。
是否暴露给公网取决于 Docker 网络和宿主机防火墙。
```

生产环境仍要配合安全组。

---

## 三、users.acl 示例

```conf
user default off
user app on >app-password ~app:* +@read +@write -flushall -flushdb -keys -config -shutdown -debug -module
user readonly on >readonly-password ~app:* +get +mget +exists +ttl
```

含义：

- 禁用默认用户。
- app 用户可读写 `app:*`。
- readonly 用户只读。

上线前要确认应用已经配置用户名和密码。

---

## 四、docker-compose.yml

```yaml
services:
  redis:
    image: redis:7
    container_name: redis-secure
    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
    volumes:
      - ./redis.conf:/usr/local/etc/redis/redis.conf:ro
      - ./users.acl:/usr/local/etc/redis/users.acl
      - ./data:/data
    networks:
      - backend
    healthcheck:
      test: ["CMD", "redis-cli", "--user", "app", "-a", "app-password", "PING"]
      interval: 10s
      timeout: 3s
      retries: 3

networks:
  backend:
    driver: bridge
```

这里没有配置 `ports`，表示宿主机外部不能直接访问。

应用容器可以通过 Compose 网络访问 `redis:6379`。

---

## 五、本地开发需要暴露端口怎么办

本地开发可以临时加：

```yaml
ports:
  - "127.0.0.1:6379:6379"
```

这表示只绑定宿主机本地地址。

比：

```yaml
ports:
  - "6379:6379"
```

更安全。

生产环境不要随意暴露端口。

---

## 六、启动和验证

启动：

```bash
docker compose up -d
```

查看状态：

```bash
docker compose ps
```

连接：

```bash
docker exec -it redis-secure redis-cli --user app -a app-password
```

测试：

```redis
PING
SET app:test ok
GET app:test
```

尝试访问不允许的 key：

```redis
GET other:test
```

应该失败。

---

## 七、数据目录

`data/` 保存持久化文件。

例如：

```text
dump.rdb
appendonlydir/
```

要注意：

- 宿主机磁盘空间。
- 目录权限。
- 备份策略。
- 不要误删数据目录。

容器删除不应该导致数据丢失。

---

## 八、健康检查

Compose healthcheck：

```yaml
healthcheck:
  test: ["CMD", "redis-cli", "--user", "app", "-a", "app-password", "PING"]
  interval: 10s
  timeout: 3s
  retries: 3
```

健康检查只能说明 Redis 能响应 `PING`。

它不能证明：

- 业务 key 正常。
- 内存充足。
- AOF 正常。
- 复制正常。

监控仍然必须有。

---

## 九、运行说明 README

建议写：

```text
# Redis 运行说明

用途：
启动命令：
连接方式：
ACL 用户：
数据目录：
持久化策略：
内存限制：
淘汰策略：
备份方式：
监控面板：
故障处理：
```

工程化的核心是让别人也能接手。

---

## 十、常见错误

### 1. 配置文件不挂载

容器启动参数散落，不易维护。

### 2. 数据目录不挂载

容器删除后数据丢失。

### 3. 生产暴露 6379 到公网

安全风险极高。

### 4. ACL 文件只读挂载

如果运行时需要 `ACL SAVE`，文件必须可写。

### 5. 健康检查替代监控

健康检查太粗，不能替代完整监控。

---

## 十一、本节练习

请完成下面练习：

1. 创建 `redis.conf`。
2. 创建 `users.acl`。
3. 编写 Docker Compose 文件。
4. 不暴露 Redis 端口，只允许内部网络访问。
5. 启动 Redis。
6. 使用 ACL 用户连接。
7. 测试只读用户不能写入。
8. 编写 Redis 运行说明 README。

---

## 十二、本节小结

这一节你学习了 Docker Compose 部署 Redis。

你需要记住：

- 配置文件、ACL 文件和数据目录要明确管理。
- 生产环境不要随意暴露 Redis 端口。
- ACL 用户要和应用配置对应。
- 数据目录要持久化并纳入备份。
- 健康检查有用，但不能替代监控。

下一节我们学习 Prometheus/Grafana 监控和核心指标。

