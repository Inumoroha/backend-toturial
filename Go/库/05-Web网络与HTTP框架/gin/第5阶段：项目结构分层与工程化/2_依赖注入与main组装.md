# 2. 依赖注入与 main 组装

本节目标：理解依赖注入的基本思想，并学会在 `main.go` 中组装 repository、service、handler 和 router。

依赖注入听起来像一个很大的概念，但在本阶段你只需要掌握一句话：

```text
一个对象需要什么依赖，不要在内部偷偷创建，而是从外部传进来。
```

---

## 一、什么是依赖

假设 `UserHandler` 需要调用 `UserService`：

```go
type UserHandler struct {
    service *service.UserService
}
```

那么 `UserService` 就是 `UserHandler` 的依赖。

再看 service：

```go
type UserService struct {
    repo *repository.UserRepository
}
```

那么 `UserRepository` 就是 `UserService` 的依赖。

依赖链大致是：

```text
UserHandler -> UserService -> UserRepository
```

---

## 二、不推荐的写法

有些初学者会这样写：

```go
func NewUserHandler() *UserHandler {
    repo := repository.NewUserRepository()
    userService := service.NewUserService(repo)

    return &UserHandler{
        service: userService,
    }
}
```

这段代码能运行，但问题很多：

- `UserHandler` 偷偷创建了 repository 和 service。
- 依赖关系藏在构造函数内部。
- 测试时很难替换 service。
- 将来 repository 需要数据库连接时，这里也要改。
- main 函数看不出项目依赖结构。

---

## 三、推荐的写法

构造函数只接收自己需要的依赖：

```go
func NewUserHandler(userService *service.UserService) *UserHandler {
    return &UserHandler{
        service: userService,
    }
}
```

service：

```go
func NewUserService(repo *repository.UserRepository) *UserService {
    return &UserService{
        repo: repo,
    }
}
```

repository：

```go
func NewUserRepository() *UserRepository {
    return &UserRepository{
        users:  make(map[uint]model.User),
        nextID: 1,
    }
}
```

然后在 `main.go` 中统一组装：

```go
func main() {
    userRepo := repository.NewUserRepository()
    userService := service.NewUserService(userRepo)
    userHandler := handler.NewUserHandler(userService)

    r := router.New(userHandler)

    r.Run(":8080")
}
```

这就是最基础的依赖注入。

---

## 四、为什么 main 适合做组装层

main 是程序入口，它天然适合做这些事情：

```text
读取配置。
初始化数据库。
初始化 Redis。
创建 repository。
创建 service。
创建 handler。
创建 router。
启动服务。
```

main 可以稍微“啰嗦”一点，因为它负责把整个项目拼起来。

但 main 不应该写：

```text
用户注册业务逻辑。
任务状态流转逻辑。
数据库查询 SQL。
JWT 解析细节。
```

---

## 五、依赖注入带来的好处

### 1. 依赖清晰

看 main 就能知道：

```text
handler 依赖 service
service 依赖 repository
repository 依赖存储
```

### 2. 测试更容易

如果 service 接收接口而不是具体 repository，测试时可以传 fake repo。

### 3. 替换实现更容易

本阶段 repository 使用内存 map。

第6阶段可以改成 GORM repository。

只要对外方法保持一致，上层代码改动就会小很多。

---

## 六、构造函数命名习惯

Go 中常见写法：

```go
func NewUserRepository() *UserRepository
func NewUserService(repo *UserRepository) *UserService
func NewUserHandler(service *UserService) *UserHandler
```

构造函数一般以 `New` 开头。

返回指针是常见做法，因为这些结构体通常包含依赖和状态。

---

## 七、组装多个模块

当项目有用户模块和任务模块时，main 可能是：

```go
func main() {
    userRepo := repository.NewUserRepository()
    taskRepo := repository.NewTaskRepository()

    userService := service.NewUserService(userRepo)
    taskService := service.NewTaskService(taskRepo)

    userHandler := handler.NewUserHandler(userService)
    taskHandler := handler.NewTaskHandler(taskService)

    r := router.New(userHandler, taskHandler)

    r.Run(":8080")
}
```

这比在各个 handler 内部偷偷创建依赖更清楚。

---

## 八、router 也通过依赖注入接收 handler

router：

```go
func New(userHandler *handler.UserHandler) *gin.Engine {
    r := gin.Default()

    api := r.Group("/api/v1")
    {
        api.GET("/users", userHandler.List)
        api.POST("/users", userHandler.Create)
        api.GET("/users/:id", userHandler.Get)
    }

    return r
}
```

如果有多个 handler：

```go
func New(userHandler *handler.UserHandler, taskHandler *handler.TaskHandler) *gin.Engine
```

---

## 九、常见问题

### 1. 构造函数里能不能创建下层依赖

小 demo 可以，但不推荐养成这个习惯。

更清晰的方式是 main 统一组装。

### 2. main 变长是不是坏事

不一定。

main 长一点可以接受，因为它是组装层。真正需要避免的是 main 中出现大量业务逻辑。

### 3. 依赖注入是不是必须用框架

不是。

Go 项目中手动依赖注入非常常见。本阶段不需要引入复杂 DI 框架。

---

## 十、本节练习

完成：

1. 给 `UserRepository` 写 `NewUserRepository`。
2. 给 `UserService` 写 `NewUserService`。
3. 给 `UserHandler` 写 `NewUserHandler`。
4. 在 `cmd/server/main.go` 中组装三者。
5. 在 `router.New` 中接收 `userHandler`。
6. 确认接口仍然能访问。

---

## 十一、本节验收

你应该能够回答：

- 什么是依赖？
- 什么是依赖注入？
- 为什么不推荐在 handler 构造函数里创建 repository？
- main 函数作为组装层有什么好处？
- Go 项目一定需要 DI 框架吗？
- 构造函数通常怎么命名？

