# 1. 使用 Docker 启动 Redis

这一节的目标很简单：不在本机直接安装 Redis，而是用 Docker 启动一个干净、可删除、可重复创建的 Redis 环境。

学完这一节后，你应该能够：

- 用 Docker 启动 Redis 容器。
- 使用 `redis-cli` 连接 Redis。
- 执行 `PING`、`SET`、`GET`、`DEL` 等基础命令。
- 理解端口映射、容器名、数据卷、密码这几个本地开发中最常见的配置。
- 知道 Redis 容器启动失败或连接不上时该怎么排查。

---

## 一、为什么用 Docker 启动 Redis

学习 Redis 有两种常见方式：

1. 直接在本机安装 Redis。
2. 使用 Docker 启动 Redis。

推荐初学阶段使用 Docker，因为它有几个好处：

- 不污染本机环境。
- 不需要手动配置系统服务。
- 可以快速删除并重建 Redis。
- 可以很方便地同时启动多个 Redis 实例。
- 后续学习主从、哨兵、集群时更容易搭环境。

Redis 本质上是一个服务进程。Docker 做的事情，就是把 Redis 和它需要的运行环境打包进一个容器里。你只需要通过端口访问这个容器即可。

---

## 二、准备工作

在开始之前，请确认本机已经安装 Docker。

打开终端，执行：

```bash
docker --version
```

如果能看到类似下面的输出，说明 Docker 已经安装：

```text
Docker version 26.x.x, build xxxxxxx
```

再执行：

```bash
docker ps
```

如果没有报错，说明 Docker 服务正在运行。

如果提示无法连接 Docker daemon，通常说明 Docker Desktop 没有启动，或者当前用户没有 Docker 权限。

---

## 三、最快方式启动 Redis

执行下面命令：

```bash
docker run -d --name redis-dev -p 6379:6379 redis:7-alpine
```

这条命令会启动一个 Redis 容器。

参数说明：

| 参数 | 含义 |
| --- | --- |
| `docker run` | 创建并启动一个容器 |
| `-d` | 后台运行容器 |
| `--name redis-dev` | 给容器取名为 `redis-dev` |
| `-p 6379:6379` | 将本机 `6379` 端口映射到容器内 `6379` 端口 |
| `redis:7-alpine` | 使用 Redis 7 的 Alpine 轻量镜像 |

Redis 默认监听端口是 `6379`。启动后，本机程序可以通过 `127.0.0.1:6379` 访问这个 Redis。

查看容器是否启动成功：

```bash
docker ps
```

正常情况下，你会看到类似信息：

```text
CONTAINER ID   IMAGE            COMMAND                  PORTS                    NAMES
xxxxxxxxxxxx   redis:7-alpine   "docker-entrypoint..."   0.0.0.0:6379->6379/tcp   redis-dev
```

查看 Redis 日志：

```bash
docker logs redis-dev
```

如果日志里能看到 Redis ready to accept connections 之类的信息，说明 Redis 已经可以接收连接。

---

## 四、使用 redis-cli 连接 Redis

Redis 官方提供了命令行工具 `redis-cli`。

如果本机没有安装 `redis-cli`，也没关系，可以直接使用容器里的 `redis-cli`：

```bash
docker exec -it redis-dev redis-cli
```

进入后，你会看到类似提示：

```text
127.0.0.1:6379>
```

先执行：

```redis
PING
```

如果返回：

```text
PONG
```

说明连接成功。

---

## 五、执行几个最基础命令

Redis 是 key-value 数据库。最基础的操作就是写入一个 key，再根据 key 读取 value。

写入一个 key：

```redis
SET name redis
```

读取这个 key：

```redis
GET name
```

返回：

```text
"redis"
```

判断 key 是否存在：

```redis
EXISTS name
```

返回：

```text
(integer) 1
```

删除 key：

```redis
DEL name
```

再次读取：

```redis
GET name
```

返回：

```text
(nil)
```

`(nil)` 表示这个 key 不存在。

---

## 六、体验过期时间

Redis 经常用来保存临时数据，比如验证码、登录 token、热点缓存。临时数据通常需要过期时间。

写入一个 10 秒后过期的 key：

```redis
SET code 123456 EX 10
```

查看剩余过期时间：

```redis
TTL code
```

返回值是剩余秒数，例如：

```text
(integer) 8
```

过十几秒后再次读取：

```redis
GET code
```

如果返回：

```text
(nil)
```

说明这个 key 已经过期并被删除。

你也可以先写入 key，再单独设置过期时间：

```redis
SET token abc
EXPIRE token 60
TTL token
```

取消过期时间：

```redis
PERSIST token
```

---

## 七、退出 redis-cli

在 `redis-cli` 中执行：

```redis
exit
```

或者按 `Ctrl + C`。

---

## 八、停止、启动和删除容器

停止 Redis 容器：

```bash
docker stop redis-dev
```

重新启动 Redis 容器：

```bash
docker start redis-dev
```

删除容器：

```bash
docker rm redis-dev
```

如果容器还在运行，需要先停止再删除：

```bash
docker stop redis-dev
docker rm redis-dev
```

也可以强制删除：

```bash
docker rm -f redis-dev
```

学习阶段可以使用强制删除，但生产环境不要随意这么做。

---

## 九、让数据保存到本机目录

前面启动的 Redis 没有显式挂载数据目录。删除容器后，容器里的数据通常也会一起消失。

如果你想让 Redis 数据保存到本机，可以挂载一个目录：

```bash
docker run -d --name redis-dev \
  -p 6379:6379 \
  -v redis-dev-data:/data \
  redis:7-alpine \
  redis-server --appendonly yes
```

Windows PowerShell 中可以写成一行：

```powershell
docker run -d --name redis-dev -p 6379:6379 -v redis-dev-data:/data redis:7-alpine redis-server --appendonly yes
```

这里新增了两个部分：

| 配置 | 含义 |
| --- | --- |
| `-v redis-dev-data:/data` | 创建并挂载 Docker volume，把 Redis 数据目录保存下来 |
| `redis-server --appendonly yes` | 启动 Redis，并开启 AOF 持久化 |

查看 Docker volume：

```bash
docker volume ls
```

这个配置适合本地学习。后面学习 Redis 持久化时，会专门讲 RDB、AOF、混合持久化和数据恢复。

---

## 十、给 Redis 设置密码

本地学习可以不设置密码，但真实项目里不要让 Redis 裸奔。

启动一个带密码的 Redis：

```bash
docker run -d --name redis-dev \
  -p 6379:6379 \
  redis:7-alpine \
  redis-server --requirepass 123456
```

Windows PowerShell 一行版：

```powershell
docker run -d --name redis-dev -p 6379:6379 redis:7-alpine redis-server --requirepass 123456
```

连接：

```bash
docker exec -it redis-dev redis-cli
```

直接执行：

```redis
PING
```

会返回类似：

```text
NOAUTH Authentication required.
```

需要先认证：

```redis
AUTH 123456
PING
```

也可以连接时直接带密码：

```bash
docker exec -it redis-dev redis-cli -a 123456
```

注意：命令行里直接写密码可能被 shell 历史记录保存。本地学习问题不大，生产环境要更谨慎。

---

## 十一、使用 Docker Compose 管理 Redis

如果你不想每次都写很长的 `docker run` 命令，可以使用 Docker Compose。

在项目目录创建 `docker-compose.yml`：

```yaml
services:
  redis:
    image: redis:7-alpine
    container_name: redis-dev
    ports:
      - "6379:6379"
    volumes:
      - redis-dev-data:/data
    command: redis-server --appendonly yes

volumes:
  redis-dev-data:
```

启动：

```bash
docker compose up -d
```

查看状态：

```bash
docker compose ps
```

查看日志：

```bash
docker compose logs -f redis
```

停止：

```bash
docker compose down
```

如果要连数据卷也一起删除：

```bash
docker compose down -v
```

`down -v` 会删除 volume，里面的数据也会被删除。执行前要确认里面没有需要保留的数据。

---

## 十二、用图形化客户端连接 Redis

除了 `redis-cli`，你也可以使用图形化客户端查看 Redis 数据。

常见客户端：

- RedisInsight
- Another Redis Desktop Manager

本地连接信息通常是：

| 配置项 | 值 |
| --- | --- |
| Host | `127.0.0.1` |
| Port | `6379` |
| Username | 留空，或者使用默认用户 |
| Password | 没设置密码就留空，设置了就填写密码 |

连接成功后，可以在图形化界面里看到 key、value、TTL、数据类型等信息。

图形化客户端适合学习和排查，但不要因此忽略 `redis-cli`。很多生产排查命令还是用命令行更直接。

---

## 十三、理解这几个概念

### 1. Redis 容器

容器里运行的是 Redis 服务进程。

你可以把它理解成一台轻量级的小机器，里面只负责运行 Redis。

### 2. 端口映射

```bash
-p 6379:6379
```

左边的 `6379` 是本机端口，右边的 `6379` 是容器内部端口。

本机程序访问 `127.0.0.1:6379`，Docker 会把请求转发给容器里的 Redis。

### 3. 容器名

```bash
--name redis-dev
```

容器名方便你后续操作：

```bash
docker logs redis-dev
docker stop redis-dev
docker exec -it redis-dev redis-cli
```

### 4. 数据卷

```bash
-v redis-dev-data:/data
```

数据卷用来保存 Redis 的数据文件。删除容器不一定会删除数据卷，所以更适合保存需要复用的数据。

### 5. Redis 数据默认在内存中

Redis 的核心特点是快，因为大部分读写操作都在内存里完成。

但内存不是硬盘。是否能在重启后恢复数据，取决于你是否开启了持久化，以及持久化配置是否正确。

---

## 十四、常见问题排查

### 1. 容器名已经存在

如果执行 `docker run` 时出现：

```text
Conflict. The container name "/redis-dev" is already in use
```

说明已经有一个叫 `redis-dev` 的容器。

查看：

```bash
docker ps -a
```

删除旧容器：

```bash
docker rm -f redis-dev
```

然后重新执行启动命令。

### 2. 6379 端口被占用

如果提示端口被占用，说明本机已经有程序在使用 `6379`。

可以换一个本机端口，例如：

```bash
docker run -d --name redis-dev -p 6380:6379 redis:7-alpine
```

这时本机连接地址变成：

```text
127.0.0.1:6380
```

容器内部 Redis 仍然监听 `6379`。

### 3. 连接不上 Redis

按下面顺序排查：

查看容器是否在运行：

```bash
docker ps
```

查看容器日志：

```bash
docker logs redis-dev
```

确认端口映射：

```bash
docker port redis-dev
```

进入容器内部测试：

```bash
docker exec -it redis-dev redis-cli PING
```

如果容器内部能 `PONG`，但本机程序连接不上，通常是端口映射、防火墙或连接地址配置问题。

### 4. 设置了密码后命令都报 NOAUTH

先认证：

```redis
AUTH 你的密码
```

或者连接时带上密码：

```bash
redis-cli -h 127.0.0.1 -p 6379 -a 你的密码
```

如果使用容器里的客户端：

```bash
docker exec -it redis-dev redis-cli -a 你的密码
```

### 5. 删除容器后数据没了

如果没有挂载 volume，也没有正确开启持久化，删除容器后数据丢失是正常的。

本地开发如果希望数据保留，请使用：

```bash
-v redis-dev-data:/data
```

并根据需要开启 AOF：

```bash
redis-server --appendonly yes
```

---

## 十五、本节练习

请自己完整操作一遍：

1. 用 Docker 启动一个名为 `redis-dev` 的 Redis 容器。
2. 使用 `docker exec -it redis-dev redis-cli` 进入 Redis。
3. 执行 `PING`，确认返回 `PONG`。
4. 执行 `SET user:1:name tom`。
5. 执行 `GET user:1:name`，确认能读取到 `tom`。
6. 执行 `SET login:code:1001 888888 EX 30`。
7. 执行 `TTL login:code:1001`，观察剩余时间。
8. 等 30 秒后再次 `GET login:code:1001`，观察结果。
9. 停止 Redis 容器。
10. 重新启动 Redis 容器。

如果你使用了持久化配置，可以观察重启后数据是否还在。

---

## 十六、本节小结

这一节你完成了 Redis 学习的第一步：把 Redis 跑起来。

你需要记住几个关键点：

- Redis 默认端口是 `6379`。
- Docker 通过 `-p 本机端口:容器端口` 暴露 Redis。
- 可以用 `docker exec -it redis-dev redis-cli` 进入 Redis 命令行。
- `PING` 返回 `PONG` 表示连接正常。
- `SET`、`GET`、`DEL` 是最基础的 key-value 操作。
- `EX`、`EXPIRE`、`TTL` 可以控制和查看过期时间。
- 删除容器可能导致数据丢失，想保留数据要使用 volume 和持久化配置。

下一节开始，我们会继续学习 Redis 的通用命令和 key 设计。真正写好 Redis，不是从记命令开始，而是从设计清晰、稳定、可维护的 key 开始。
