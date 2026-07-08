# 5. 阶段练习：Todo API 测试清单

本节目标：给 Todo API 补一组有价值的测试，而不是只测一个成功路径。

---

## 一、Handler 测试清单

建议至少覆盖：

```text
POST /todos
  - 创建成功返回 201
  - JSON 格式错误返回 400
  - title 为空返回 400

GET /todos/{id}
  - 查询存在 Todo 返回 200
  - id 非数字返回 400
  - id 不存在返回 404

PATCH /todos/{id}
  - 更新 title 成功
  - 更新 done 成功
  - title 为空返回 400
  - id 不存在返回 404

DELETE /todos/{id}
  - 删除成功返回 204
  - id 不存在返回 404
```

---

## 二、Middleware 测试清单

建议至少覆盖：

```text
auth
  - 无 token 返回 401
  - 错误 token 返回 401
  - 正确 token 调用 next

requestID
  - 没有 X-Request-ID 时自动生成
  - 有 X-Request-ID 时透传
  - 响应 Header 中包含 X-Request-ID

recovery
  - panic 返回 500
  - 普通请求正常透传

cors
  - OPTIONS 返回 204
  - 响应中包含 CORS Header
```

---

## 三、集成测试清单

用 `httptest.NewServer` 测一个完整流程：

```text
创建 Todo
-> 查询 Todo
-> 更新 Todo
-> 列表中能看到 Todo
-> 删除 Todo
-> 再查询返回 404
```

这个测试能证明路由、Handler、存储和 JSON 处理至少在主流程上是通的。

---

## 四、运行测试

```bash
go test ./...
```

检查 data race：

```bash
go test -race ./...
```

如果你使用了内存 map，`-race` 非常重要。它能帮助你发现并发读写问题。

---

## 五、阶段复盘

完成本阶段后，请写下：

- 哪些测试是 Handler 单元测试？
- 哪些测试是 Middleware 测试？
- 哪些测试是集成测试？
- 为什么错误路径必须测试？
- 为什么 `resp.Body.Close()` 不能忘？

测试阶段完成后，你已经具备了维护 HTTP 服务的基础安全感。下一阶段进入 HTTP Client，学习如何可靠调用第三方 API。

