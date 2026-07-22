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

---

## 融合补充

+### 05-01 MySQL、GORM、模型与连接池

#### 安装与连接

```bash
go get gorm.io/gorm gorm.io/driver/mysql
```

DSN 只来自配置或环境变量：

```go
func openDB(dsn string) (*gorm.DB, error) {
    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
    if err != nil { return nil, err }
    sqlDB, err := db.DB()
    if err != nil { return nil, err }
    sqlDB.SetMaxOpenConns(20)
    sqlDB.SetMaxIdleConns(10)
    sqlDB.SetConnMaxLifetime(time.Hour)
    if err := sqlDB.PingContext(context.Background()); err != nil { return nil, err }
    return db, nil
}
```

连接池参数要结合数据库最大连接数、应用副本数和负载观测调整。启动时 Ping 能及早发现 DSN、网络或账号问题；运行中每个查询仍要使用请求 Context 和超时。

#### Model 设计

```go
type User struct {
    ID uint `gorm:"primaryKey" json:"id"`
    Email string `gorm:"size:120;not null;uniqueIndex" json:"email"`
    Name string `gorm:"size:50;not null" json:"name"`
    PasswordHash string `gorm:"size:100;not null" json:"-"`
    Role string `gorm:"size:20;not null;default:user" json:"role"`
    CreatedAt time.Time `json:"created_at"`
    UpdatedAt time.Time `json:"updated_at"`
    DeletedAt gorm.DeletedAt `gorm:"index" json:"-"`
}
```

`json:"-"` 防止密码哈希被序列化。唯一性必须在数据库中建立唯一索引，仅靠“先查再写”无法抵抗并发竞争。`gorm.DeletedAt` 提供软删除，普通查询会自动过滤已删除记录。

#### CRUD 的关键边界

```go
err := db.WithContext(ctx).Create(&user).Error
err := db.WithContext(ctx).First(&user, "id = ?", id).Error
err := db.WithContext(ctx).Model(&user).Updates(map[string]interface{}{"name": in.Name}).Error
err := db.WithContext(ctx).Delete(&user).Error
```

`First` 的 `ErrRecordNotFound` 要转换为领域 `ErrNotFound`。局部更新使用 DTO 白名单，不要把客户端 JSON 直接映射为整行 `Save`，否则零值、权限字段或密码字段可能被意外覆盖。软删除数据的恢复、审计和永久清理要显式使用 `Unscoped`，不要让普通用户接口看到它们。

开发环境可以使用 `AutoMigrate` 迭代模型；生产环境使用版本化迁移，迁移在发布流程中执行并具备备份和回滚方案。

**验收：**能连接数据库、创建和读取用户；密码哈希不在 JSON 中；重复邮箱触发唯一约束；删除后的记录默认不再出现在列表中。

+### 05-02 Repository、分页、事务、迁移与排错

#### Repository 只处理存取

```go
func (r *GormTaskRepository) List(ctx context.Context, ownerID uint, page, pageSize int) ([]domain.Task, int64, error) {
    base := r.db.WithContext(ctx).Model(&domain.Task{}).Where("owner_id = ?", ownerID)
    var total int64
    if err := base.Count(&total).Error; err != nil { return nil, 0, err }
    var tasks []domain.Task
    offset := (page - 1) * pageSize
    err := base.Order("created_at desc").Offset(offset).Limit(pageSize).Find(&tasks).Error
    return tasks, total, err
}
```

查询必须有稳定排序，否则在数据变化时分页会出现重复或遗漏。排序字段来自白名单；`Order`、表名和列名不是参数占位符可安全处理的值，绝不能直接使用用户输入。

#### 事务由 Service 决定边界

一个业务操作同时改任务状态并写事件时，需要原子性：

```go
func (s *TaskService) Complete(ctx context.Context, taskID, userID uint) error {
    return s.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        var task domain.Task
        if err := tx.First(&task, "id = ?", taskID).Error; err != nil { return err }
        if task.OwnerID != userID { return domain.ErrForbidden }
        if task.Status == domain.TaskDone { return nil } // 幂等
        if err := tx.Model(&task).Update("status", domain.TaskDone).Error; err != nil { return err }
        return tx.Create(&domain.TaskEvent{TaskID: task.ID, Type: "completed"}).Error
    })
}
```

事务内避免网络调用、长时间文件处理和等待用户输入。它们会长时间占有连接与锁。业务层决定一个用例是否需要事务，Repository 接受 tx 或封装最小的读写操作。

#### 迁移与数据库错误

- 开发：`AutoMigrate` 方便验证模型。
- 生产：使用版本化 migration 文件，提交前在副本数据演练，备份后执行，保留回滚脚本。
- 重复键：根据驱动错误识别并映射为 `ErrConflict`。
- 记录不存在：映射为 `ErrNotFound`。
- 超时与不可用：记录详细日志，客户端接收通用失败，不泄露 DSN 或 SQL。

排错从 DSN、DNS、端口、账户、容器网络、迁移版本、连接池指标、慢 SQL 和 `EXPLAIN` 依次检查。不要用无限增大连接数解决慢查询或锁竞争。

**验收：**分页返回 total；并发重复创建不会产生两条记录；事务中任一步失败后所有写入都回滚；迁移可从空库和已有库重复验证。

