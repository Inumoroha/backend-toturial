# 5. 阶段练习：给 Todo 加入观测能力

本节目标：给 Todo API 增加结构化日志、pprof 和基础压测验证。

---

## 一、需求

完成：

- 使用 `slog` 初始化 logger。
- 请求日志使用结构化字段。
- 每条请求日志包含 Request ID。
- 单独启动 `localhost:6060` pprof 服务。
- 用 `hey` 压测 `/health` 和 `/todos`。

---

## 二、建议日志字段

```text
request_id
method
path
status
duration_ms
remote_addr
user_agent
```

错误日志额外包含：

```text
error
```

---

## 三、压测命令

```bash
hey -n 1000 -c 50 http://localhost:8080/health
hey -n 1000 -c 50 http://localhost:8080/todos
```

观察：

- 是否有 500。
- P95/P99 延迟。
- 日志是否包含所有字段。
- pprof 页面是否可访问。

---

## 四、阶段复盘

完成本阶段后，请写下：

- 你的请求日志包含哪些字段？
- 如何通过 request_id 关联错误日志？
- pprof 监听在哪个地址？
- 压测结果中的 P95 和 P99 是多少？
- 如果接口突然变慢，你会先看什么？

下一阶段进入源码阅读和原理，把前面使用过的抽象往标准库内部看一层。

