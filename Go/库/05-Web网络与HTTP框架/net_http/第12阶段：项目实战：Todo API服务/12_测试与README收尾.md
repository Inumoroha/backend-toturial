# 12. 测试与 README 收尾

本节目标：给 Todo API 补齐基本测试和项目说明，让它成为一个可交付的小项目。

---

## 一、测试范围

至少包含：

- Service 测试。
- Handler 测试。
- Middleware 测试。
- 一个完整流程集成测试。

建议优先顺序：

```text
先测 Service 业务规则
-> 再测 Handler 状态码和 JSON
-> 再测 Middleware 是否拦截或透传
-> 最后测完整 HTTP 流程
```

运行：

```bash
go test ./...
go test -race ./...
```

---

## 二、README 应包含什么

```markdown
# Todo API

## 启动

go run ./cmd/server

## 接口

GET /health
GET /todos
POST /todos
GET /todos/{id}
PATCH /todos/{id}
DELETE /todos/{id}

## curl 示例
```

README 要让别人能快速启动和验证项目。

---

## 三、最小测试清单

Service：

```text
Create title 为空 -> ErrInvalidTitle
Create 正常 -> 返回 ID
Get 不存在 -> ErrNotFound
Update title 为空 -> ErrInvalidTitle
Delete 不存在 -> ErrNotFound
```

Handler：

```text
POST /todos 成功 -> 201
POST /todos 非法 JSON -> 400
GET /todos/{id} 非法 ID -> 400
GET /todos/{id} 不存在 -> 404
DELETE /todos/{id} 成功 -> 204
```

Middleware：

```text
Auth 无 Token -> 401 且不调用 next
Auth 正确 Token -> 调用 next
Recovery 捕获 panic -> 500
RequestID 写入响应 Header
```

---

## 四、curl 示例

```bash
curl -i http://localhost:8080/health

curl -i -X POST http://localhost:8080/todos `
  -H "Content-Type: application/json" `
  -H "Authorization: Bearer dev-token" `
  -d "{\"title\":\"learn net/http\"}"

curl -i http://localhost:8080/todos
```

---

## 五、项目复盘

完成后请总结：

- 代码分了哪些层？
- 每层职责是什么？
- 哪些中间件是全局的？
- 哪些路由需要鉴权？
- 哪些测试最有价值？

下一阶段进入短链接服务，它会加入重定向、短码生成和访问统计等更真实的后端场景。
