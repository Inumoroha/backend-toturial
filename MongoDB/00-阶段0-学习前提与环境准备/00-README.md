# 第 0 阶段：学习前提与环境准备

本阶段的目标不是立刻深入 MongoDB，而是把学习 MongoDB + Go 后端开发所需要的基础环境、知识结构和练习方法准备好。

如果第 0 阶段做得扎实，后面学习 CRUD、索引、聚合、事务、复制集和分片时会轻松很多；如果跳过这一步，后面常见的问题会变成“代码没错但环境连不上”“知道语法但不知道为什么这样设计”“会写查询但不会判断性能”。

## 本阶段你要完成什么

完成本阶段后，你应该具备：

- 能说清楚 MongoDB 在后端系统中解决什么问题。
- 能在本机启动 MongoDB。
- 能使用 `mongosh` 连接 MongoDB。
- 能使用 MongoDB Compass 查看数据。
- 能准备好 Go 开发环境。
- 能理解后续教程中经常出现的 Go、HTTP、数据库基础概念。
- 能完成一次从环境启动、连接、创建数据库、插入测试数据、查询数据的完整闭环。

## 文件学习顺序

请按下面顺序学习：

| 顺序 | 文件 | 作用 |
| --- | --- | --- |
| 00 | `00-README.md` | 本阶段总览 |
| 01 | `01-学习目标与知识地图.md` | 建立整体认知，知道后面要学什么 |
| 02 | `02-开发工具准备.md` | 安装和验证 Go、Git、VS Code、Docker 等工具 |
| 03 | `03-MongoDB本地环境搭建.md` | 用 Docker 启动本地 MongoDB |
| 04 | `04-mongosh与Compass入门.md` | 学会命令行和图形化连接工具 |
| 05 | `05-Go后端前置知识.md` | 补齐学习 MongoDB Go Driver 所需 Go 基础 |
| 06 | `06-数据库基础前置知识.md` | 补齐索引、事务、一致性等数据库基础 |
| 07 | `07-后端工程基础.md` | 补齐 API、配置、日志、测试、分层等工程知识 |
| 08 | `08-阶段验收与下一步.md` | 阶段练习、检查清单和进入第 1 阶段的标准 |

## 建议学习时间

如果你已有 Go 和 Docker 基础：

- 预计 1-2 天完成。

如果你是刚转向 Go 后端：

- 建议 3-5 天完成。
- 不要急着跳到 CRUD。MongoDB 的很多问题不是语法问题，而是工程习惯和数据建模问题。

## 推荐学习方式

每读一个文件，做三件事：

1. 运行里面的命令。
2. 记录遇到的错误。
3. 用自己的话写一句总结。

建议新建一个个人笔记文件：

```text
notes/
  00-阶段0-学习笔记.md
```

笔记模板：

```md
# 第 0 阶段学习笔记

## 今天学了什么

## 我运行过的命令

## 遇到的问题

## 我现在还不理解的点

## 明天继续做什么
```

## 官方资料入口

后续学习以官方资料为主，中文博客可以作为辅助，但不要把二手教程当唯一依据。

- MongoDB Manual：<https://www.mongodb.com/docs/manual/>
- MongoDB Go Driver：<https://www.mongodb.com/docs/drivers/go/current/>
- MongoDB Shell：<https://www.mongodb.com/docs/mongodb-shell/>
- MongoDB Compass：<https://www.mongodb.com/docs/compass/current/>
- MongoDB Docker Hub：<https://hub.docker.com/_/mongo>
- Go 官方文档：<https://go.dev/doc/>

