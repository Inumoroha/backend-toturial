# 0. 测试 net/http 服务总览

本阶段目标：学会不用手动启动服务，也能测试 Handler、Middleware 和 HTTP Client。

核心包：

```go
net/http/httptest
```

手动 `go run` 加 `curl` 很适合学习，但它不能替代自动化测试。真实项目需要在每次修改后快速确认：

- 状态码是否正确。
- JSON 响应是否正确。
- 错误路径是否正确。
- 中间件是否拦截或透传。
- HTTP Client 是否正确处理第三方响应。

这就是 `httptest` 的价值。

---

## 一、本阶段学习顺序

1. `httptest.NewRequest`。
2. `httptest.NewRecorder`。
3. Handler 单元测试。
4. Middleware 测试。
5. `httptest.NewServer` 集成测试。
6. Todo API 测试实践。

---

## 二、本阶段测试分层

建议把测试分成三类：

```text
Handler 测试：直接调用 Handler，检查状态码和响应体。
Middleware 测试：检查是否调用 next，是否写 Header，是否提前返回。
集成测试：用 httptest.NewServer 跑完整 HTTP 流程。
```

不要一开始所有测试都走完整服务。越底层的行为，用越小的测试越清楚。

---

## 三、本阶段达标标准

完成本阶段后，你应该能：

- 构造测试请求。
- 捕获 Handler 响应。
- 解析 JSON 响应并断言字段。
- 测试错误路径。
- 测试中间件是否调用 next。
- 使用 `httptest.NewServer` 测完整流程或模拟第三方 API。
- 运行 `go test ./...` 和 `go test -race ./...`。
