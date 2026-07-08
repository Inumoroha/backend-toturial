# 0. 数据库、GORM 与业务数据持久化总览

本阶段目标：把前面保存在内存 map 中的数据，替换为数据库中的持久化数据。你会学习如何在 Gin 项目中接入 GORM，如何定义模型，如何完成 CRUD、分页、软删除、唯一索引、事务和数据库错误处理。

前面阶段为了专注 Gin，我们一直使用内存 map：

```go
var users = map[uint]User{}
```

这种方式适合入门，但它有明显问题：

- 服务重启后数据丢失。
- 多个服务实例之间无法共享数据。
- 并发读写 map 有风险。
- 无法使用索引、事务、复杂查询。
- 无法支撑真实业务。

本阶段要解决的问题是：

```text
让用户数据真正保存到数据库中。
```

---

## 一、本阶段你会学到什么

本阶段覆盖这些内容：

```text
使用 Docker 或本地数据库准备 MySQL。
理解 DSN。
使用 GORM 连接数据库。
配置数据库连接池。
定义 GORM 模型。
使用 AutoMigrate 创建表。
使用 Create / First / Find / Save / Delete。
实现分页、排序、筛选。
处理唯一索引冲突。
处理 gorm.ErrRecordNotFound。
使用事务。
把 repository 从内存 map 改成 GORM。
```

---

## 二、为什么选 GORM

Go 操作数据库有多种方式：

```text
database/sql
sqlx
GORM
ent
SQLC
```

本教程使用 GORM，原因是：

- Gin 初学项目中非常常见。
- CRUD 写法直观。
- 支持模型、关联、事务、软删除。
- 学习成本比手写 SQL 低。
- 适合先建立后端项目完整闭环。

但你也要知道：GORM 不是数据库能力的替代品。后续仍然要学习 SQL、索引、事务和执行计划。

---

## 三、本阶段默认技术选择

本阶段默认：

```text
数据库：MySQL 8
ORM：GORM
驱动：gorm.io/driver/mysql
项目结构：第5阶段的 handler/service/repository 分层
```

如果你使用 PostgreSQL，思路一样，只需要换驱动和 DSN：

```bash
go get gorm.io/driver/postgres
```

本教程会以 MySQL 为主讲，因为 Gin + GORM + MySQL 是 Go Web 入门中非常常见的组合。

---

## 四、本阶段推荐学习顺序

建议按下面顺序：

```text
连接数据库与连接池
→ GORM 模型、CRUD、分页与软删除
→ 事务、唯一索引与数据库错误处理
→ Repository 设计实践
→ 分页排序筛选实践
→ 数据库排错清单
→ 综合实践：数据库版用户模块
→ 接口测试请求集合
```

这样安排是因为：

- 先连接数据库，否则后面都无法运行。
- 再定义模型和基本 CRUD。
- 再处理真实业务中的错误和事务。
- 最后把第5阶段的 repository 替换为 GORM 实现。

---

## 五、本阶段最终目标

本阶段结束后，用户模块应该从：

```text
handler -> service -> memory repository -> map
```

变成：

```text
handler -> service -> GORM repository -> MySQL
```

接口路径保持不变：

```text
GET    /api/v1/users
POST   /api/v1/users
GET    /api/v1/users/:id
PUT    /api/v1/users/:id
PATCH  /api/v1/users/:id/status
DELETE /api/v1/users/:id
```

但数据来源变成数据库。

验收标准：

- 服务重启后，已经创建的用户仍然存在。
- 邮箱唯一由数据库索引保护。
- 用户不存在能返回 404。
- 邮箱重复能返回 409。
- 列表接口支持分页、排序、筛选。
- 删除用户使用软删除。

---

## 六、本阶段项目目录

建议在第5阶段基础上增加：

```text
internal/database/
  mysql.go
internal/model/
  user.go
internal/repository/
  user_repository.go
```

典型结构：

```text
gin-layer-demo/
  cmd/server/main.go
  configs/config.yaml
  internal/database/mysql.go
  internal/model/user.go
  internal/repository/user_repository.go
  internal/service/user_service.go
  internal/handler/user_handler.go
  internal/router/router.go
```

---

## 七、本阶段需要特别注意

数据库阶段最容易出现这些问题：

- DSN 写错。
- 数据库没启动。
- 数据库名不存在。
- Docker 容器内还用 `127.0.0.1` 连接 MySQL。
- 忘记 `parseTime=True`。
- AutoMigrate 没执行。
- 唯一索引冲突被当成 500。
- 查询不到数据时没有处理 `gorm.ErrRecordNotFound`。
- 分页没有限制 `page_size`。

这些问题后面都会逐步展开。

