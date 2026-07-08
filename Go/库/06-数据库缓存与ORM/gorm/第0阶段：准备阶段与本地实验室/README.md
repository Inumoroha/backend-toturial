# 00 准备阶段

本阶段目标：把学习 GORM 前必须具备的环境和基础补齐。GORM 是 Go 的 ORM 工具，但它最终操作的是数据库，所以你不能只学 GORM API，也要理解 SQL、表结构、索引、事务和连接池。

## 学习顺序

1. [开发环境准备](./01-environment-setup.md)
2. [SQL 与数据库基础](./02-sql-database-basics.md)
3. [学习方法与练习规范](./03-learning-workflow.md)
4. [完整本地练习环境搭建](./04-complete-local-lab.md)
5. [使用 Docker Compose 管理 MySQL 学习环境](./05-Docker-Compose管理MySQL.md)
6. [使用客户端连接并观察数据库](./06-客户端连接与数据库观察.md)
7. [Go 项目环境变量与配置](./07-Go项目环境变量与配置.md)
8. [环境问题排查手册](./08-环境问题排查手册.md)

## 本阶段你要完成什么

- 安装 Go 并理解 `go mod`。
- 准备 MySQL 或 PostgreSQL。
- 能手写基础 SQL。
- 能创建一个最小 Go 项目。
- 能用命令行连接数据库。
- 建立“看 GORM 生成 SQL”的学习习惯。

## 推荐选择

如果你刚开始学后端，推荐使用：

- 数据库：MySQL 8.x
- Web 框架：Gin
- 配置方式：环境变量优先
- API 调试：Apifox 或 Postman

后续教程里的示例默认偏向 MySQL，但 GORM 的大部分 API 对 PostgreSQL、SQLite 也适用。
