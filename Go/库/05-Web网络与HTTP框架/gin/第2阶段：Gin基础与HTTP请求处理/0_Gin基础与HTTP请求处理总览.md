# 0. Gin 基础与 HTTP 请求处理总览

本阶段目标：从标准库 HTTP 服务过渡到 Gin，掌握 Gin 的启动方式、路由注册、请求参数读取、JSON 响应和内存版 CRUD。

第一阶段你已经知道标准库中一次请求大概这样处理：

```text
http.HandleFunc 注册路由
handler 读取 *http.Request
handler 通过 http.ResponseWriter 写响应
```

Gin 做的事情不是改变 HTTP 本质，而是让这些操作更简洁：

```text
router.GET 注册路由
*gin.Context 统一封装请求和响应
c.JSON 快速返回 JSON
c.Param / c.Query / c.ShouldBindJSON 读取参数
```

---

## 一、本阶段要解决的问题

本阶段解决这些问题：

- 如何安装 Gin。
- `gin.Default()` 和 `gin.New()` 有什么区别。
- 如何注册 GET、POST、PUT、DELETE 路由。
- 如何使用路由组 `/api/v1`。
- 如何读取路径参数、查询参数和 JSON 请求体。
- 如何返回 JSON 响应。
- 如何实现一个不依赖数据库的用户 CRUD。

---

## 二、本阶段统一练习接口

本阶段围绕用户资源练习：

```text
GET    /ping
GET    /api/v1/users
GET    /api/v1/users/:id
POST   /api/v1/users
PUT    /api/v1/users/:id
DELETE /api/v1/users/:id
```

暂时不接数据库，使用内存 map 保存数据。

这样做的原因是：先把 Gin 的 HTTP 处理学清楚，再去接数据库。不要一开始把路由、参数、GORM、认证混在一起。

---

## 三、本阶段推荐文件结构

本阶段先保持简单：

```text
gin-lab/
  go.mod
  go.sum
  main.go
  requests.http
```

等进入项目分层阶段，再拆成 `handler`、`service`、`repository`。

初学阶段过早拆目录，反而容易看不清 Gin 的核心流程。

---

## 四、本阶段学习顺序

建议按下面顺序：

```text
安装 Gin
→ 启动第一个 Gin 服务
→ 理解 gin.Default
→ 注册路由和路由组
→ 读取参数
→ 实现内存版 CRUD
→ 使用 requests.http 完整测试
```

---

## 五、本阶段完成标准

完成本阶段后，你应该能够：

- 独立创建 Gin 服务。
- 写出 `/ping` 接口。
- 使用 `/api/v1` 路由组。
- 读取 `:id` 路径参数。
- 读取 `?page=1` 查询参数。
- 使用 `ShouldBindJSON` 读取请求体。
- 实现用户 CRUD。
- 能根据 400、404、405 判断常见问题。

