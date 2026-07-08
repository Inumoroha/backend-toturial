# 11 main 函数组装全流程

本节目标：从 `main.go` 的角度把配置、数据库、Repository、Service、Handler、Router 全部组装起来。工程化学习不能只看单个层，还要知道项目启动时这些对象如何连接。

---

## 一、目标结构

```text
cmd/server/main.go
internal/config
internal/database
internal/repository
internal/service
internal/handler
internal/middleware
```

`main.go` 的职责是组装，不写业务逻辑。

---

## 二、main 基本流程

```text
1. 加载配置
2. 初始化数据库
3. 执行迁移
4. 初始化 Repository
5. 初始化 Service
6. 初始化 Handler
7. 注册路由
8. 启动 HTTP 服务
```

---

## 三、代码示例

```go
func main() {
    cfg, err := config.Load()
    if err != nil {
        log.Fatal(err)
    }

    db, err := database.New(cfg.DB)
    if err != nil {
        log.Fatal(err)
    }

    if err := database.Migrate(db); err != nil {
        log.Fatal(err)
    }

    userRepo := repository.NewUserRepository(db)
    articleRepo := repository.NewArticleRepository(db)
    categoryRepo := repository.NewCategoryRepository(db)
    tagRepo := repository.NewTagRepository(db)
    commentRepo := repository.NewCommentRepository(db)

    authService := service.NewAuthService(userRepo, cfg.JWT)
    articleService := service.NewArticleService(db, articleRepo, categoryRepo, tagRepo)
    commentService := service.NewCommentService(commentRepo, articleRepo)

    authHandler := handler.NewAuthHandler(authService)
    articleHandler := handler.NewArticleHandler(articleService)
    commentHandler := handler.NewCommentHandler(commentService)

    r := gin.Default()

    api := r.Group("/api")
    authHandler.RegisterRoutes(api)

    protected := api.Group("")
    protected.Use(middleware.Auth(cfg.JWT.Secret))
    articleHandler.RegisterRoutes(protected)
    commentHandler.RegisterRoutes(protected)

    addr := ":" + cfg.App.Port
    if err := r.Run(addr); err != nil {
        log.Fatal(err)
    }
}
```

---

## 四、组装原则

### 1. 依赖从外向内传

数据库对象创建一次，然后传给 Repository。

### 2. Service 依赖 Repository

Service 不自己创建 Repository。

### 3. Handler 依赖 Service

Handler 不直接依赖 GORM。

### 4. main 不写业务

main 只负责连接对象。

---

## 五、常见错误

### 1. 在 Handler 中初始化 Service

这会让依赖关系混乱。

### 2. 在 Repository 中读取配置

Repository 只应该接收已经创建好的 `*gorm.DB`。

### 3. main 里写 SQL

迁移可以调用函数，业务 SQL 不应该出现在 main。

---

## 六、本节达标标准

学完本节后，你应该能够做到：

- 写出清晰的 main 启动流程。
- 正确组装 Repository、Service、Handler。
- 知道每层依赖谁。
- 避免 main 函数变成业务垃圾场。

---

## 颗粒度补强：本节应该如何真正练会

这一节不要只停留在“看懂概念”。建议你按下面的方式把「main函数组装全流程」练成可以在项目中使用的能力。

### 1. 先写最小可运行示例

在 `code/gorm-study` 中准备一个最小 Go 程序，只保留和本节主题直接相关的模型、数据库连接和 GORM 调用。不要一开始就放进完整项目结构，否则你很难判断问题来自 GORM、分层、路由还是参数绑定。

建议流程：

```text
1. 定义最小模型
2. AutoMigrate 建表
3. 插入 2 到 3 条测试数据
4. 执行本节核心 GORM API
5. 打印 SQL 日志
6. 到数据库客户端里验证结果
```

### 2. 必须观察 SQL

学习 GORM 的关键不是背 API，而是看它生成的 SQL。执行本节代码时，建议临时开启：

```go
db = db.Debug()
```

或者在初始化时使用：

```go
Logger: logger.Default.LogMode(logger.Info)
```

你至少要回答：

```text
这段 GORM 代码生成了几条 SQL？
SQL 中有没有 WHERE 条件？
有没有 LIMIT 或 ORDER BY？
有没有使用软删除条件 deleted_at IS NULL？
更新或删除时 RowsAffected 是多少？
```

### 3. 用数据库客户端验证

不要只相信 Go 程序输出。每次运行后，都在 MySQL 客户端中执行对应查询，例如：

```sql
SHOW TABLES;
SHOW CREATE TABLE users;
SHOW INDEX FROM users;
SELECT * FROM users ORDER BY id DESC LIMIT 10;
```

如果本节涉及文章、评论、订单或标签，就去查对应表：

```sql
SELECT * FROM articles ORDER BY id DESC LIMIT 10;
SELECT * FROM comments ORDER BY id DESC LIMIT 10;
SELECT * FROM orders ORDER BY id DESC LIMIT 10;
SELECT * FROM article_tags ORDER BY article_id DESC LIMIT 10;
```

### 4. 主动制造一个错误

每节都建议故意制造一个小错误，然后观察报错和数据变化。比如：

```text
把数据库密码写错，观察连接错误。
把唯一字段重复插入，观察唯一索引错误。
把查询 ID 改成不存在的值，观察 ErrRecordNotFound。
把更新条件去掉，观察 GORM 是否阻止全表更新。
把事务里的 return err 去掉，观察是否错误提交。
```

你真正排查过错误，才算把这个知识点学稳。

### 5. 放回真实项目场景

最后把本节知识放回博客系统或订单系统中思考：

```text
博客系统里哪个接口会用到这个知识点？
订单系统里哪个流程会因为这个知识点写错而出问题？
这个知识点应该放在 Model、Repository、Service 还是 Handler？
它需要测试吗？需要看 SQL 日志吗？需要事务吗？
```

如果你能回答这些问题，说明你已经从“知道 API”进入到“能做后端项目”的层次。

### 6. 本节复盘问题

学完后请写下这几个答案：

```text
1. 本节最核心的 GORM API 是什么？
2. 它生成的 SQL 大致是什么？
3. 它最容易踩的坑是什么？
4. 如果不用 GORM，原生 SQL 会怎么写？
5. 在博客系统或订单系统中，它会出现在哪个模块？
```
