# 1. 用 Docker 启动 Redis 并创建 Go 项目

本节目标：你将能在本机启动一个 Redis，并创建一个可以运行 Redis 示例代码的 Go 项目。

简短引入：真实后端项目里，Redis 通常不是单独学习命令，而是跟 Go 服务、数据库和业务接口一起工作。第一步先把环境跑通，后面所有示例都建立在这个基础上。

## 一、为什么需要它

可以理解为，Redis 是后端服务旁边的一个高速工具箱。用户资料缓存、验证码、短链接、库存扣减、排行榜都可能用到它。

学习阶段推荐用 Docker，是因为它能把 Redis 版本、端口、启动方式固定下来。以后你换电脑或给同事演示，也更容易复现。

```text
学习 Redis-Go 时，不要先纠结生产集群，先保证本地环境稳定可重复。
```

## 二、基本用法

新建教程练习目录：

Windows PowerShell：

```powershell
mkdir redis-go-practice
cd redis-go-practice
```

Linux/macOS：

```bash
mkdir redis-go-practice
cd redis-go-practice
```

创建 `docker-compose.yml`：

```yaml
services:
  redis:
    image: redis:7
    container_name: redis-go-practice
    ports:
      - "6379:6379"
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis-data:/data

volumes:
  redis-data:
```

启动 Redis：

Windows PowerShell：

```powershell
docker compose up -d
docker compose ps
```

Linux/macOS：

```bash
docker compose up -d
docker compose ps
```

测试 Redis：

```powershell
docker exec -it redis-go-practice redis-cli ping
```

预期输出：

```text
PONG
```

初始化 Go 项目：

```powershell
go mod init redis-go-practice
go get github.com/redis/go-redis/v9
```

Linux/macOS 同样可以执行：

```bash
go mod init redis-go-practice
go get github.com/redis/go-redis/v9
```

## 三、关键参数/语法/代码结构

`ports: "6379:6379"` 表示把容器里的 Redis 暴露到本机 6379 端口。Go 程序连接 `localhost:6379` 就能访问。

`appendonly yes` 开启 AOF 持久化。学习阶段这样做有一个好处：你重启容器后，部分数据不会立刻丢失。但真实项目里是否开启、怎么配置，要由运维策略、磁盘、恢复目标决定。

`go get github.com/redis/go-redis/v9` 安装 Go 客户端。真实项目中要把依赖版本提交到 `go.mod` 和 `go.sum`，不要只依赖本机缓存。

## 四、真实后端场景示例

假设你要做一个用户服务：

- 数据库保存用户主资料。
- Redis 缓存用户昵称、头像、权限摘要。
- HTTP 接口先查 Redis，未命中再查数据库。

本节还不写业务代码，只是搭建这条链路中 Redis 的本地环境。后面你会逐步把它接进 Go 服务里。

## 五、注意点

学习环境可以无密码，但生产环境通常必须开启认证、网络隔离和访问控制。

本地端口 6379 如果被占用，可以改成：

```yaml
ports:
  - "6380:6379"
```

然后 Go 连接地址改成 `localhost:6380`。

如果 Docker 没有启动，Go 代码会报连接失败。不要急着改代码，先确认 Redis 容器是否正常运行。

## 六、常见误区

- 误区：把 Docker 容器删了还以为数据一定保留。  
  原因：数据是否保留取决于 volume 是否还在。

- 误区：一上来就连接云 Redis。  
  原因：网络、权限、费用和安全策略会干扰基础学习。

- 误区：只会用 `redis-cli`，不会让 Go 程序连接。  
  原因：后端工程师真正要掌握的是 Redis 在应用代码里的使用方式。

## 七、本节达标标准

- 能用 Docker Compose 启动 Redis。
- 能通过 `redis-cli ping` 得到 `PONG`。
- 能初始化 Go module。
- 能安装 `github.com/redis/go-redis/v9`。
- 能说明本地 Redis 端口和 Go 连接地址的关系。

