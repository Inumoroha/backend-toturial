# 1. RESTful 路由设计与资源命名

本节目标：学会用资源名和 HTTP 方法设计接口，而不是把动作全部写进路径。完成本节后，你应该能根据一个简单业务模块设计出清晰、稳定、容易维护的 API 路径。

---

## 一、为什么要先讲路由设计

很多初学者写接口时，会自然写出这种路径：

```text
/getUsers
/getUserById
/createUser
/updateUser
/deleteUser
```

这些接口能用，但不适合长期维护。原因是：

- 路径里混入太多动作。
- HTTP 方法没有发挥作用。
- 资源关系不清楚。
- 接口文档看起来混乱。
- 前后端协作时不容易形成统一习惯。

更推荐的设计是：

```text
GET    /api/v1/users
POST   /api/v1/users
GET    /api/v1/users/:id
PUT    /api/v1/users/:id
DELETE /api/v1/users/:id
```

这就是 RESTful 风格的基本直觉：

```text
路径表示资源，HTTP 方法表示动作。
```

---

## 二、什么是资源

资源可以理解为系统中的业务对象。

常见资源：

```text
users       用户
tasks       任务
articles    文章
comments    评论
orders      订单
products    商品
```

资源名通常使用复数：

```text
/users
/tasks
/articles
```

因为这些路径代表的是一组资源，而不是单个固定对象。

单个资源通过 ID 定位：

```text
/users/1
/tasks/99
/articles/1001
```

---

## 三、HTTP 方法表达动作

RESTful API 中，动作主要由 HTTP 方法表达。

| 方法 | 常见含义 | 示例 |
| --- | --- | --- |
| `GET` | 查询资源 | `GET /api/v1/users` |
| `POST` | 创建资源 | `POST /api/v1/users` |
| `PUT` | 整体更新资源 | `PUT /api/v1/users/1` |
| `PATCH` | 局部更新资源 | `PATCH /api/v1/users/1/status` |
| `DELETE` | 删除资源 | `DELETE /api/v1/users/1` |

例如创建用户，不需要写：

```text
POST /api/v1/createUser
```

因为 `POST` 已经表达了“创建”的动作，所以路径写资源名即可：

```text
POST /api/v1/users
```

---

## 四、用户模块路由设计

本阶段用户模块推荐路由：

```text
GET    /api/v1/users
POST   /api/v1/users
GET    /api/v1/users/:id
PUT    /api/v1/users/:id
PATCH  /api/v1/users/:id/status
DELETE /api/v1/users/:id
```

逐个解释：

```text
GET /users
查询用户列表。

POST /users
创建一个新用户。

GET /users/:id
查询某个用户详情。

PUT /users/:id
整体更新某个用户。

PATCH /users/:id/status
只修改用户状态。

DELETE /users/:id
删除某个用户。
```

这里的 `:id` 是路径参数，在 Gin 中可以通过：

```go
c.Param("id")
```

读取。

---

## 五、PUT 和 PATCH 的区别

`PUT` 更偏向整体更新。

例如：

```http
PUT /api/v1/users/1
Content-Type: application/json

{
  "username": "alice-new",
  "email": "alice_new@example.com",
  "age": 22,
  "status": "active"
}
```

这表示客户端提交了用户的新完整状态。

`PATCH` 更偏向局部更新。

例如：

```http
PATCH /api/v1/users/1/status
Content-Type: application/json

{
  "status": "disabled"
}
```

这表示只修改状态，不关心其他字段。

初学阶段不必陷入教条，但要形成基本习惯：完整更新用 `PUT`，局部字段更新用 `PATCH`。

---

## 六、路径参数和查询参数的边界

路径参数用于定位资源。

例如：

```text
/api/v1/users/1
```

这里的 `1` 是用户这个资源的身份。

查询参数用于筛选、搜索、分页。

例如：

```text
/api/v1/users?page=1&page_size=10&status=active&keyword=alice
```

这些参数不是某个用户的身份，而是列表查询条件。

不要把分页写成：

```text
/api/v1/users/page/1/page_size/10
```

这样会让路径变得很难维护。

---

## 七、API 版本号

推荐使用：

```text
/api/v1
```

完整示例：

```text
GET /api/v1/users
```

为什么要有版本号？

- 方便未来新增不兼容版本。
- 方便前端逐步迁移。
- 方便网关、Nginx、日志系统识别 API。

例如未来要调整用户响应结构，可以保留：

```text
/api/v1/users
```

同时新增：

```text
/api/v2/users
```

---

## 八、Gin 中注册这些路由

示例：

```go
func main() {
    r := gin.Default()

    api := r.Group("/api/v1")
    {
        api.GET("/users", listUsers)
        api.POST("/users", createUser)
        api.GET("/users/:id", getUser)
        api.PUT("/users/:id", updateUser)
        api.PATCH("/users/:id/status", updateUserStatus)
        api.DELETE("/users/:id", deleteUser)
    }

    r.Run(":8080")
}
```

注意：

```go
api.PATCH("/users/:id/status", updateUserStatus)
```

这条路由比直接写：

```go
api.POST("/users/:id/disable", disableUser)
```

更统一，因为它仍然表达的是“更新用户资源的一部分”。

---

## 九、常见反例

### 1. 动词堆在路径里

不推荐：

```text
/api/v1/getAllUsers
/api/v1/createNewUser
/api/v1/deleteUserById
```

推荐：

```text
GET /api/v1/users
POST /api/v1/users
DELETE /api/v1/users/:id
```

### 2. 方法和语义不匹配

不推荐：

```text
GET /api/v1/users/delete/1
```

删除操作不应该使用 `GET`，因为 GET 应该是安全的、只读的。

推荐：

```text
DELETE /api/v1/users/1
```

### 3. 查询条件放进路径

不推荐：

```text
/api/v1/users/status/active/page/1
```

推荐：

```text
/api/v1/users?status=active&page=1
```

---

## 十、本节练习

请为博客系统设计路由：

业务需求：

- 创建文章。
- 查询文章列表。
- 查询文章详情。
- 更新文章。
- 删除文章。
- 发布文章。
- 查询某篇文章的评论列表。
- 给某篇文章发表评论。

参考设计：

```text
POST   /api/v1/articles
GET    /api/v1/articles
GET    /api/v1/articles/:id
PUT    /api/v1/articles/:id
DELETE /api/v1/articles/:id
PATCH  /api/v1/articles/:id/status
GET    /api/v1/articles/:id/comments
POST   /api/v1/articles/:id/comments
```

思考：

- 评论是独立资源，还是文章的子资源？
- 发布文章是否可以理解为修改文章状态？
- 查询列表时分类、标签、关键词应该放哪里？

---

## 十一、本节验收

你应该能够回答：

- RESTful 路由为什么尽量使用名词？
- HTTP 方法在接口设计中承担什么职责？
- `PUT` 和 `PATCH` 的区别是什么？
- 为什么分页和筛选参数应该放在 query string？
- `/api/v1` 的意义是什么？
- 为什么不要用 GET 执行删除操作？


