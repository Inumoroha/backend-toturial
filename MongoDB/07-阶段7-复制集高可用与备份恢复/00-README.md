# 第 7 阶段：复制集、高可用与备份恢复

前面你已经学习了 CRUD、Go Driver、建模、聚合、索引和事务。本阶段开始进入 MongoDB 的运行形态：复制集、高可用、读写关注、故障切换、备份恢复和 Change Streams。

如果说前 6 阶段主要解决“怎么写对代码”，第 7 阶段解决的是：

```text
数据库节点挂了怎么办？
写入到底确认到哪里才算成功？
能不能从 secondary 读？
怎么备份和恢复？
怎么监听数据变化？
```

## 本阶段目标

完成本阶段后，你应该能做到：

- 解释 primary、secondary、arbiter 的作用。
- 理解 oplog、复制和选举的基本机制。
- 用 Docker Compose 搭建本地三节点复制集。
- 停掉 primary 后观察重新选主。
- 理解 readPreference 的收益和风险。
- 理解 writeConcern: `w:1` 和 `majority` 的差异。
- 使用 `mongodump` 和 `mongorestore` 做一次备份恢复。
- 理解 Atlas Backup 的定位。
- 使用 Change Streams 监听订单状态变化。
- 判断重要业务为什么应该考虑 majority 写入。

## 学习文件顺序

| 顺序 | 文件 | 作用 |
| --- | --- | --- |
| 00 | `00-README.md` | 本阶段总览 |
| 01 | `01-复制集基本概念.md` | primary、secondary、arbiter、oplog |
| 02 | `02-Docker-Compose搭建三节点复制集.md` | 本地搭建复制集 |
| 03 | `03-选举与故障切换实验.md` | 停掉 primary 并观察选主 |
| 04 | `04-readPreference读偏好.md` | primary/secondary 读的收益和风险 |
| 05 | `05-writeConcern写关注与majority.md` | w:1、majority 和重要写入 |
| 06 | `06-readConcern与一致性.md` | 读取一致性基础 |
| 07 | `07-备份恢复-mongodump与mongorestore.md` | 本地备份恢复演练 |
| 08 | `08-Atlas-Backup与生产备份策略.md` | 生产环境备份策略 |
| 09 | `09-Change-Streams基础.md` | 监听订单状态变更 |
| 10 | `10-Go中配置读写关注与监听变更.md` | Go Driver v2 配置示例 |
| 11 | `11-阶段练习-复制集故障与恢复演练.md` | 综合练习 |
| 12 | `12-阶段验收与高可用清单.md` | 验收和清单 |

## 本阶段提醒

本阶段会启动多个 MongoDB 容器，端口会和第 0 阶段的单节点容器冲突。

如果你已有：

```powershell
docker ps
```

看到 `mongodb-learning` 占用 `27017`，复制集实验前可以先停止：

```powershell
docker stop mongodb-learning
```

实验结束后再启动：

```powershell
docker start mongodb-learning
```

## 官方资料

本阶段主要参考：

- Replication：<https://www.mongodb.com/docs/manual/replication/>
- Replica Set Members：<https://www.mongodb.com/docs/manual/core/replica-set-members/>
- Replica Set Elections：<https://www.mongodb.com/docs/manual/core/replica-set-elections/>
- Read Preference：<https://www.mongodb.com/docs/manual/core/read-preference/>
- Write Concern：<https://www.mongodb.com/docs/manual/reference/write-concern/>
- Read Concern：<https://www.mongodb.com/docs/manual/reference/read-concern/>
- MongoDB Database Tools：<https://www.mongodb.com/docs/database-tools/>
- Change Streams：<https://www.mongodb.com/docs/manual/changeStreams/>
- Go Driver Change Streams：<https://www.mongodb.com/docs/drivers/go/current/fundamentals/crud/read-operations/change-streams/>

