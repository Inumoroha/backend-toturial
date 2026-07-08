# 03. MongoDB 本地环境搭建

本节使用 Docker 启动一个本地 MongoDB。这样做的好处是干净、可重复、方便删除和重建，不会把系统环境弄乱。

## 1. 本地 MongoDB 的目标

我们要启动一个 MongoDB 容器：

- 容器名称：`mongodb-learning`
- 端口：`27017`
- 管理员用户名：`root`
- 管理员密码：`example`
- 数据持久化到 Docker volume

最终连接 URI：

```text
mongodb://root:example@localhost:27017/?authSource=admin
```

## 2. 拉取 MongoDB 镜像

执行：

```powershell
docker pull mongo:latest
```

查看镜像：

```powershell
docker images mongo
```

说明：

- `mongo:latest` 适合学习阶段。
- 生产环境不要随便使用 `latest`，应该固定具体版本。
- 学习 MongoDB 语法时，使用新版本通常更合适。

## 3. 创建数据卷

创建一个 Docker volume：

```powershell
docker volume create mongodb-learning-data
```

查看：

```powershell
docker volume ls
```

为什么要用 volume？

如果不挂载数据卷，容器删除后数据也可能丢失。学习阶段虽然可以重建，但使用 volume 更接近真实环境。

## 4. 启动 MongoDB 容器

执行：

```powershell
docker run -d `
  --name mongodb-learning `
  -p 27017:27017 `
  -e MONGO_INITDB_ROOT_USERNAME=root `
  -e MONGO_INITDB_ROOT_PASSWORD=example `
  -v mongodb-learning-data:/data/db `
  mongo:latest
```

注意：PowerShell 多行命令使用反引号 `` ` `` 续行。不要把它写成 Linux/macOS 里的反斜杠 `\`。

如果你想写成一行：

```powershell
docker run -d --name mongodb-learning -p 27017:27017 -e MONGO_INITDB_ROOT_USERNAME=root -e MONGO_INITDB_ROOT_PASSWORD=example -v mongodb-learning-data:/data/db mongo:latest
```

## 5. 检查容器状态

执行：

```powershell
docker ps
```

应该能看到：

```text
mongodb-learning
0.0.0.0:27017->27017/tcp
```

查看日志：

```powershell
docker logs mongodb-learning
```

如果日志最后没有明显报错，通常说明启动成功。

持续查看日志：

```powershell
docker logs -f mongodb-learning
```

退出持续日志查看：按 `Ctrl + C`。

## 6. 使用 mongosh 连接

执行：

```powershell
mongosh "mongodb://root:example@localhost:27017/?authSource=admin"
```

连接成功后，你会进入 `mongosh` 交互环境。

查看数据库：

```javascript
show dbs
```

查看当前数据库：

```javascript
db
```

切换或创建学习数据库：

```javascript
use mongodb_learning
```

插入一条数据：

```javascript
db.env_check.insertOne({
  name: "stage-0",
  status: "ok",
  created_at: new Date()
})
```

查询：

```javascript
db.env_check.find()
```

格式化查询：

```javascript
db.env_check.find().pretty()
```

退出：

```javascript
exit
```

## 7. 理解连接 URI

连接 URI：

```text
mongodb://root:example@localhost:27017/?authSource=admin
```

拆开看：

| 部分 | 含义 |
| --- | --- |
| `mongodb://` | MongoDB 普通连接协议 |
| `root` | 用户名 |
| `example` | 密码 |
| `localhost` | 主机地址 |
| `27017` | MongoDB 默认端口 |
| `authSource=admin` | 到 `admin` 数据库进行认证 |

为什么要写 `authSource=admin`？

因为我们用 `MONGO_INITDB_ROOT_USERNAME` 和 `MONGO_INITDB_ROOT_PASSWORD` 创建的是 root 用户，这个用户默认创建在 `admin` 数据库。你连接业务库时，也需要告诉 MongoDB 去哪里认证。

## 8. 常用 Docker 命令

停止容器：

```powershell
docker stop mongodb-learning
```

启动已停止的容器：

```powershell
docker start mongodb-learning
```

重启容器：

```powershell
docker restart mongodb-learning
```

查看容器：

```powershell
docker ps
docker ps -a
```

查看日志：

```powershell
docker logs mongodb-learning
```

进入容器：

```powershell
docker exec -it mongodb-learning bash
```

在容器内使用 mongosh：

```bash
mongosh "mongodb://root:example@localhost:27017/?authSource=admin"
```

退出容器：

```bash
exit
```

## 9. 删除并重建环境

如果你只是想删除容器，但保留数据：

```powershell
docker stop mongodb-learning
docker rm mongodb-learning
```

之后可以用前面的 `docker run` 命令重新创建容器，数据还在 volume 里。

如果你想彻底删除数据：

```powershell
docker stop mongodb-learning
docker rm mongodb-learning
docker volume rm mongodb-learning-data
```

注意：删除 volume 会清空数据库数据。学习阶段可以这么做，真实项目不要随便执行。

## 10. 端口冲突处理

如果启动容器时报错类似：

```text
Bind for 0.0.0.0:27017 failed: port is already allocated
```

说明本机 `27017` 已被占用。

查看已有容器：

```powershell
docker ps
```

也可以换成本机端口 `27018`：

```powershell
docker run -d `
  --name mongodb-learning `
  -p 27018:27017 `
  -e MONGO_INITDB_ROOT_USERNAME=root `
  -e MONGO_INITDB_ROOT_PASSWORD=example `
  -v mongodb-learning-data:/data/db `
  mongo:latest
```

此时连接 URI 变成：

```text
mongodb://root:example@localhost:27018/?authSource=admin
```

## 11. 认证失败处理

如果连接时报：

```text
Authentication failed
```

优先检查：

- 用户名是不是 `root`。
- 密码是不是 `example`。
- URI 有没有写 `authSource=admin`。
- 容器是不是用旧 volume 启动的。

如果你第一次用了其他密码，后来只改 `docker run` 的环境变量，旧 volume 不会自动修改已有 root 密码。

学习阶段最简单的重置方式：

```powershell
docker stop mongodb-learning
docker rm mongodb-learning
docker volume rm mongodb-learning-data
docker volume create mongodb-learning-data
```

然后重新执行启动命令。

## 12. 本节验收

你必须完成：

```powershell
docker ps
mongosh "mongodb://root:example@localhost:27017/?authSource=admin"
```

并能在 `mongosh` 中执行：

```javascript
use mongodb_learning
db.env_check.insertOne({ name: "stage-0", status: "ok", created_at: new Date() })
db.env_check.find().pretty()
```

如果你能看到刚插入的数据，本地 MongoDB 环境就搭好了。

