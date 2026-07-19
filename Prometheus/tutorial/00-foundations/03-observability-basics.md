# 可观测性基础与 SLO

Prometheus 学习的起点不是指标类型，而是“哪些变化值得被观察和响应”。

## 1. Logs、Metrics、Traces 的分工

| 信号 | 回答的问题 | 典型内容 |
| --- | --- | --- |
| Logs | 某个事件发生了什么 | 请求失败原因、订单 ID、错误堆栈 |
| Metrics | 一段时间内系统整体怎样 | QPS、错误率、P95、队列深度 |
| Traces | 一次请求经过了哪里 | API -> 库存 -> 支付的耗时分解 |

不要把所有上下文塞进 Metrics。`order_id` 对一条日志或 trace 很有用，但会让指标产生无界时间序列。

## 2. RED 与 USE

对用户请求采用 RED：

- Rate：单位时间请求数。
- Errors：失败请求数或错误比例。
- Duration：延迟分布，优先 Histogram。

对资源和组件采用 USE：

- Utilization：资源利用率。
- Saturation：排队、阻塞、连接池耗尽等饱和信号。
- Errors：资源或组件错误。

练习：为订单服务填写：

| 问题 | 观测信号 | 可能指标 |
| --- | --- | --- |
| 请求变多了吗？ | Rate | `http_requests_total` |
| 用户是否失败？ | Errors | `http_requests_total{status=~"5.."}` |
| 请求是否变慢？ | Duration | `http_request_duration_seconds` |
| Worker 是否积压？ | Saturation | `order_queue_depth` |
| 下游是否超时？ | Errors/Duration | `dependency_requests_total`、Histogram |

## 3. 从 SLI 推导 SLO

一个简单的可用性 SLI：

```text
成功请求数 / 总请求数
```

如果 30 天 SLO 是 99.9%，错误预算约为：

```text
30 天总分钟数 × 0.1%
```

不要直接把所有 5xx 都当作用户错误。先定义：

- 哪些 route 纳入 SLO。
- 哪些状态码算失败。
- 健康检查和内部探针是否排除。
- 请求量太小时如何解释比率。
- 窗口是 5 分钟、1 小时还是 30 天。

## 4. 指标设计表

在写代码前填写：

| 字段 | 示例 |
| --- | --- |
| 指标名 | `http_request_duration_seconds` |
| 类型 | Histogram |
| 单位 | seconds |
| Label | `method`, `route`, `status` |
| 不使用的字段 | URL 参数、用户 ID、错误文本 |
| 采集时机 | handler 完成后 |
| 主要查询 | P95、SLO 达标率 |
| 主要告警 | 5xx 比例、P95 超阈值 |
| 责任团队 | backend |
| Runbook | `/runbooks/order-api.md` |

## 5. 设计前的五个问题

1. 这个指标能否直接支持一个决策？
2. Label 的值域是否有限、稳定、可枚举？
3. 指标重启后归零是否可以接受？
4. 是否需要跨实例聚合？
5. 用户看到异常时，指标能否帮助缩小范围？

## 6. 阶段练习

为 `POST /orders`、库存依赖、异步 Worker 分别设计指标。至少包含：

- 3 个 Counter。
- 2 个 Histogram。
- 2 个 Gauge。
- 每个指标的单位、label、用途和禁止字段。
- 一条从 SLO 推导的告警。

不要先实现。下一阶段先观察 Prometheus 的抓取和数据模型，再在 Go 中实现这些设计。

## 7. 从 SLO 推导一组最小指标

假设 `POST /orders` 的目标是 30 天可用性 99.9%，且 5xx 算作失败：

```text
允许失败比例 = 1 - 0.999 = 0.001
30 天错误预算约为 43.2 分钟
```

从这个目标可以推导：

```text
分子：rate(http_requests_total{route="/orders",status=~"5.."}[5m])
分母：rate(http_requests_total{route="/orders"}[5m])
结果：错误比例，单位 ratio
```

延迟 SLO 如果是 99% 请求小于 500ms，则必须有 `le="0.5"` 的 Histogram bucket；否则只能计算近似 P99，不能直接计算达标率。这个推导过程要写进指标设计表，防止实现时遗漏关键 bucket。

## 8. 阶段验收

- [ ] 能区分一条日志、一个指标和一条 trace 各自适合回答的问题。
- [ ] 能为订单 API 写出 Rate、Errors、Duration 三类指标。
- [ ] 能从一个 99.9% SLO 推导错误比例和错误预算。
- [ ] 能说明哪些字段必须留在日志/Tracing，而不能成为 label。
- [ ] 能为每个候选指标写出单位、统计边界和空数据语义。
