# 0. Todo API 项目总览

本阶段目标：把前面学过的 `net/http` 基础能力串成一个完整项目。Todo API 虽小，但足够训练后端工程的基本闭环。

前面阶段大多是在单点学习：会写 Handler、会写 Middleware、会配置 Server、会测试。项目阶段要做的是把这些能力按工程顺序串起来。

---

## 一、本项目最终能力

项目会包含：

- 项目结构拆分。
- 配置加载。
- `http.Server` 启动和优雅关闭。
- 路由注册。
- Middleware 链。
- JSON 请求响应。
- 内存存储版本。
- 可选数据库版本。
- Handler 和 Middleware 测试。
- README 和 curl 示例。

---

## 二、建议目录

建议目录：

```text
todo-api/
  cmd/
    server/
      main.go
  internal/
    handler/
    middleware/
    model/
    service/
    store/
  go.mod
  README.md
```

---

## 三、本阶段文件顺序

建议按下面顺序完成：

1. `01_需求分析与接口设计.md`
2. `02_项目结构与模块划分.md`
3. `03_Model_Service_Store设计.md`
4. `04_路由Handler与响应封装.md`
5. `05_中间件与Server启动.md`
6. `06_初始化Go模块与目录.md`
7. `07_实现todo包_model_store_service.md`
8. `08_实现httpapi响应与Handler.md`
9. `09_实现middleware链.md`
10. `10_组装main与优雅关闭.md`
11. `11_接口验证与错误场景.md`
12. `12_测试与README收尾.md`

前 1-5 节偏设计，后 6-12 节偏落地实现。这样安排是为了先知道要做什么，再一步步写出来。

---

## 四、最小闭环

项目最小闭环不是一次写完所有接口，而是先跑通：

```text
启动服务
-> GET /health 返回 ok
-> POST /todos 创建 Todo
-> GET /todos 查询列表
-> 错误请求能返回 JSON 错误
```

最小闭环跑通后，再补详情、更新、删除、中间件、测试和 README。

---

## 五、达标标准

完成本阶段后，你应该能：

- 从零创建一个 Go HTTP API 项目。
- 解释每个目录的职责。
- 写出内存 Store，并保证并发安全。
- 使用 Service 封装业务规则。
- 使用 Handler 处理 HTTP 输入输出。
- 使用 Middleware 处理日志、Request ID、Recovery、鉴权。
- 使用 `http.Server` 启动并优雅关闭。
- 写出 curl 验证命令和基础测试。
