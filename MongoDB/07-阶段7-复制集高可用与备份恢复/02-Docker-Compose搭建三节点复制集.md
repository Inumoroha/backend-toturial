# 02. Docker Compose 搭建三节点复制集

本节用 Docker Compose 搭建本地三节点复制集。它用于学习，不是生产部署模板。

## 1. 准备目录

建议在练习区创建：

```powershell
mkdir C:\Users\HP\Desktop\mongodb-rs-lab
cd C:\Users\HP\Desktop\mongodb-rs-lab
```

## 2. 端口规划

```text
mongo1 -> localhost:27017
mongo2 -> localhost:27018
mongo3 -> localhost:27019
```

如果第 0 阶段的 `mongodb-learning` 正在占用 27017，先停止：

```powershell
docker stop mongodb-learning
```

## 3. docker-compose.yml

创建 `docker-compose.yml`：

```yaml
services:
  mongo1:
    image: mongo:latest
    container_name: mongo-rs1
    command: ["mongod", "--replSet", "rs0", "--bind_ip_all"]
    ports:
      - "27017:27017"
    volumes:
      - mongo1-data:/data/db

  mongo2:
    image: mongo:latest
    container_name: mongo-rs2
    command: ["mongod", "--replSet", "rs0", "--bind_ip_all"]
    ports:
      - "27018:27017"
    volumes:
      - mongo2-data:/data/db

  mongo3:
    image: mongo:latest
    container_name: mongo-rs3
    command: ["mongod", "--replSet", "rs0", "--bind_ip_all"]
    ports:
      - "27019:27017"
    volumes:
      - mongo3-data:/data/db

volumes:
  mongo1-data:
  mongo2-data:
  mongo3-data:
```

启动：

```powershell
docker compose up -d
```

查看：

```powershell
docker ps
```

## 4. 初始化复制集

连接 mongo1：

```powershell
mongosh "mongodb://localhost:27017"
```

执行：

```javascript
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo1:27017" },
    { _id: 1, host: "mongo2:27017" },
    { _id: 2, host: "mongo3:27017" }
  ]
})
```

查看状态：

```javascript
rs.status()
```

等待其中一个节点成为 PRIMARY，其他为 SECONDARY。

## 5. 外部连接串

容器内部使用 `mongo1:27017`、`mongo2:27017`、`mongo3:27017`。

宿主机连接可能使用：

```text
mongodb://localhost:27017,localhost:27018,localhost:27019/?replicaSet=rs0
```

如果出现主机名解析问题，可以在 Windows hosts 文件中加入：

```text
127.0.0.1 mongo1
127.0.0.1 mongo2
127.0.0.1 mongo3
```

hosts 文件路径：

```text
C:\Windows\System32\drivers\etc\hosts
```

修改 hosts 需要管理员权限。

也可以只在容器内部用 `mongosh` 实验，减少本机解析问题。

## 6. 写入测试

连接 primary 后：

```javascript
use rs_stage7
db.health.insertOne({ name: "replica-set-ok", created_at: new Date() })
db.health.find().pretty()
```

在 secondary 上读需要设置：

```javascript
db.getMongo().setReadPref("secondary")
```

或者连接串配置读偏好。学习阶段先在后面章节再实验。

## 7. 查看节点角色

```javascript
rs.status().members.map(m => ({
  name: m.name,
  stateStr: m.stateStr
}))
```

也可以：

```javascript
db.hello()
```

查看当前节点是否 primary。

## 8. 停止和清理

停止：

```powershell
docker compose down
```

停止并删除数据卷：

```powershell
docker compose down -v
```

注意：`-v` 会删除复制集数据。

## 9. 常见问题

### 27017 端口冲突

停止第 0 阶段容器：

```powershell
docker stop mongodb-learning
```

### rs.initiate host 写 localhost

不推荐在容器复制集成员里写：

```javascript
host: "localhost:27017"
```

因为容器内部的 localhost 指向自己，不是其他容器。

### 节点一直不是 SECONDARY

检查日志：

```powershell
docker logs mongo-rs1
docker logs mongo-rs2
docker logs mongo-rs3
```

检查网络和成员 host 是否能互相解析。

## 10. 本节验收

你应该能做到：

- 启动三个 MongoDB 容器。
- 初始化复制集。
- 查看 primary 和 secondary。
- 插入一条测试数据。
- 停止并清理实验环境。

