# 0. HTTP Client 总览

本阶段目标：学会用 Go 标准库调用第三方 HTTP API，并写出可靠的客户端封装。

后端工程师不只写服务端，也经常调用：

- 支付接口。
- 短信接口。
- OAuth 服务。
- 内部微服务。
- 第三方开放 API。

HTTP Client 写错很常见，典型问题包括：

- 没有设置超时。
- 没有带 context。
- 忘记关闭响应体。
- 把 404 当成 `client.Do` 的 error。
- 每次请求都新建 `http.Client`。
- 不理解连接池和 Transport。

本阶段就是专门把这些坑补上。

---

## 一、本阶段学习顺序

1. `http.Client` 基础。
2. `http.NewRequestWithContext`。
3. 超时和取消。
4. 正确关闭响应体。
5. 错误状态码处理。
6. `http.Transport` 和连接池。
7. 使用 `httptest.NewServer` 测试客户端。

---

## 二、本阶段核心流程

一个可靠的 Client 调用大致是：

```text
接收 ctx
-> 构造 request
-> 设置 Header
-> client.Do
-> defer resp.Body.Close
-> 检查状态码
-> 解析 JSON
-> 返回业务结果
```

只要少一步，就可能埋下线上问题。

---

## 三、本阶段达标标准

完成本阶段后，你应该能：

- 封装一个可复用的 `http.Client`。
- 使用 `NewRequestWithContext`。
- 设置全局超时和单次调用超时。
- 正确处理非 2xx 状态码。
- 正确关闭响应体。
- 解释 Client 和 Transport 的关系。
- 用 `httptest.NewServer` 测试自己的 Client。
