# 2. 使用图形化客户端连接 Redis

上一节我们已经用 Docker 启动了 Redis，并通过 `redis-cli` 执行了几个基础命令。

这一节我们换一个视角：使用图形化客户端查看 Redis 里的数据。图形化客户端不是必须的，但它很适合初学阶段建立直觉：你能直接看到 key、value、数据类型、TTL 和数据库编号。

学完这一节后，你应该能够：

- 使用图形化客户端连接本地 Redis。
- 查看 Redis 中的 key、value 和 TTL。
- 新增、修改、删除简单数据。
- 理解图形化客户端适合做什么，不适合做什么。
- 知道连接失败时该从哪里排查。

---

## 一、先确认 Redis 已经启动

如果你还没有启动 Redis，可以先执行：

```bash
docker run -d --name redis-dev -p 6379:6379 redis:7-alpine
```

如果容器已经存在，可以直接启动：

```bash
docker start redis-dev
```

确认容器正在运行：

```bash
docker ps
```

再测试一下 Redis 是否可用：

```bash
docker exec -it redis-dev redis-cli PING
```

如果返回：

```text
PONG
```

说明 Redis 正常。

---

## 二、常见图形化客户端

学习阶段可以选择下面任意一个工具：

| 客户端 | 特点 |
| --- | --- |
| RedisInsight | Redis 官方工具，功能完整，适合学习和排查 |
| Another Redis Desktop Manager | 界面简洁，连接和查看数据很方便 |

如果你只是刚开始学习 Redis，任选一个即可。不要把时间花在比较工具上，先把连接、查看、修改数据这几个动作跑通更重要。

---

## 三、连接本地 Redis

如果你使用上一节的命令启动 Redis：

```bash
docker run -d --name redis-dev -p 6379:6379 redis:7-alpine
```

那么图形化客户端里通常这样填写：

| 配置项 | 值 |
| --- | --- |
| Host | `127.0.0.1` |
| Port | `6379` |
| Username | 留空，或者填写 `default` |
| Password | 没设置密码就留空 |
| Database | `0` |

Redis 默认有多个逻辑数据库，本地学习时默认使用 `0` 号数据库即可。

如果你启动 Redis 时换过本机端口，比如：

```bash
docker run -d --name redis-dev -p 6380:6379 redis:7-alpine
```

那么图形化客户端里端口要填写：

```text
6380
```

注意：`-p 6380:6379` 左边是本机端口，右边是容器内部端口。客户端连接的是本机端口。

---

## 四、如果 Redis 设置了密码

如果你用下面方式启动 Redis：

```bash
docker run -d --name redis-dev -p 6379:6379 redis:7-alpine redis-server --requirepass 123456
```

那么图形化客户端里要填写密码：

| 配置项 | 值 |
| --- | --- |
| Host | `127.0.0.1` |
| Port | `6379` |
| Username | 留空，或者填写 `default` |
| Password | `123456` |

如果密码不对，通常会出现 `NOAUTH`、`WRONGPASS`、`Authentication failed` 一类错误。

---

## 五、先用命令行写入几条数据

为了让图形化客户端里有东西可看，先进入 `redis-cli`：

```bash
docker exec -it redis-dev redis-cli
```

写入几条数据：

```redis
SET user:1:name tom
SET user:1:email tom@example.com
SET article:1001:title "Redis Docker Tutorial"
SET login:code:1001 888888 EX 60
```

查看 key：

```redis
GET user:1:name
TTL login:code:1001
```

然后打开图形化客户端刷新，就应该能看到这些 key。

---

## 六、在图形化客户端里观察什么

打开连接后，重点观察这几件事。

### 1. key 的命名

你会看到类似：

```text
user:1:name
user:1:email
article:1001:title
login:code:1001
```

这种用冒号分隔的命名方式很常见。它不是真的目录，只是一种约定，用来让 key 更容易分类和搜索。

### 2. value 的内容

点击 key 后，可以看到对应 value。

比如：

```text
user:1:name -> tom
```

Redis 最底层是 key-value 模型。无论后面学 String、Hash、List、Set、ZSet，本质上都是用 key 找到一份 value，只是 value 的结构不同。

### 3. 数据类型

现在写入的这些 key 都是 String 类型。

后续你学习 Hash、List、Set、ZSet 时，图形化客户端会显示不同的数据类型，也会提供不同的查看方式。

### 4. TTL

`login:code:1001` 设置了 60 秒过期时间。

你可以在图形化客户端里观察它的 TTL 逐渐减少，最后 key 自动消失。

这个现象非常重要。Redis 经常用来保存临时数据，TTL 是缓存、验证码、登录态、限流计数里很常用的能力。

---

## 七、在图形化客户端里新增和删除 key

大多数图形化客户端都支持新增 key。

你可以手动新增一个 key：

| 配置项 | 值 |
| --- | --- |
| Key | `debug:message` |
| Type | String |
| Value | `hello redis` |
| TTL | 不设置，或者设置 120 秒 |

保存后，回到命令行验证：

```bash
docker exec -it redis-dev redis-cli GET debug:message
```

如果能读到：

```text
"hello redis"
```

说明图形化客户端写入成功。

删除 key 后，也可以用命令行验证：

```bash
docker exec -it redis-dev redis-cli EXISTS debug:message
```

如果返回：

```text
(integer) 0
```

说明 key 已不存在。

---

## 八、图形化客户端适合做什么

图形化客户端适合：

- 本地学习时观察数据。
- 查看某个 key 的类型、value、TTL。
- 手动修改少量测试数据。
- 排查缓存内容是否符合预期。
- 观察不同数据结构的形态。

比如你在 Go 项目里写了文章缓存，可以用图形化客户端确认：

- key 是否真的写入 Redis。
- key 名称是否符合预期。
- value 是否是正确的 JSON。
- TTL 是否设置成功。

---

## 九、图形化客户端不适合做什么

图形化客户端不适合：

- 在生产环境随意搜索大量 key。
- 执行危险命令，比如 `FLUSHALL`、`FLUSHDB`。
- 替代自动化脚本或程序逻辑。
- 长时间实时监控高流量 Redis。
- 在不了解影响的情况下批量删除 key。

尤其要注意：生产环境不要随手使用 `KEYS *` 搜索全库。Redis 是单线程处理命令的，某些慢操作可能影响其他请求。

学习阶段可以大胆试，但生产环境要克制。

---

## 十、连接失败怎么排查

### 1. 检查容器是否运行

```bash
docker ps
```

如果看不到 `redis-dev`，说明容器没运行。

启动它：

```bash
docker start redis-dev
```

### 2. 检查端口映射

```bash
docker port redis-dev
```

正常会看到类似：

```text
6379/tcp -> 0.0.0.0:6379
```

如果你看到的是 `6380`，客户端也要连接 `6380`。

### 3. 用容器里的 redis-cli 测试

```bash
docker exec -it redis-dev redis-cli PING
```

如果这里都失败，说明 Redis 容器内部就有问题。

### 4. 检查密码

如果 Redis 设置了密码，客户端必须填写密码。

可以用命令行验证：

```bash
docker exec -it redis-dev redis-cli -a 123456 PING
```

如果返回 `PONG`，说明密码正确。

### 5. 检查连接地址

本机 Docker 启动的 Redis，通常连接：

```text
127.0.0.1
```

不要填写容器 ID，也不要填写 `redis-dev`，除非你是在另一个 Docker 容器里通过 Docker 网络访问它。

---

## 十一、本节练习

请完成下面练习：

1. 启动本地 Redis 容器。
2. 使用图形化客户端连接 `127.0.0.1:6379`。
3. 用 `redis-cli` 写入 `user:1:name`、`user:1:email`。
4. 在图形化客户端里查看这两个 key。
5. 写入一个带 TTL 的验证码 key。
6. 在图形化客户端里观察 TTL 变化。
7. 在图形化客户端里新增 `debug:message`。
8. 用 `redis-cli` 读取 `debug:message`。
9. 删除 `debug:message`。
10. 用 `EXISTS` 确认它已经不存在。

---

## 十二、本节小结

这一节你学会了用图形化客户端连接 Redis。

你需要记住：

- 本地 Redis 通常连接 `127.0.0.1:6379`。
- 如果 Docker 端口映射是 `6380:6379`，客户端要连 `6380`。
- 图形化客户端能直观看到 key、value、类型和 TTL。
- 图形化客户端适合学习和排查，不适合在生产环境随意批量操作。
- 命令行和图形化客户端应该配合使用，而不是只依赖其中一个。

下一节我们会进一步理解 Redis 的核心：key-value 模型。
