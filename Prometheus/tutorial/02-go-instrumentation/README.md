# 阶段 2：Go 埋点与指标设计

这一阶段把阶段 0 的服务变成一个可以被 Prometheus 解释的服务。重点不是“调用一个函数就有指标”，而是让指标的语义、label、单位和统计时机都正确。

## 学习目标

- 理解 `client_golang` 的 Registry、Collector、MetricVec 和 promhttp。
- 为 HTTP 服务实现 RED 指标。
- 为依赖调用、队列和业务结果设计指标。
- 使用路由模板控制 label 基数。
- 测试指标名称、类型、label 和样本值。
- 保护 `/metrics`，避免把管理端点暴露给不可信流量。

## 教程顺序

1. [指标类型、命名和基数](01-metric-types-naming-cardinality.md)
2. [Go client 与 HTTP 埋点](02-go-client-http-instrumentation.md)
3. [Middleware、依赖指标和指标测试](03-middleware-dependency-testing.md)

## 目标指标集

先实现下面这组稳定指标：

```text
order_service_http_requests_total{method,route,status}
order_service_http_request_duration_seconds{method,route}
order_service_http_in_flight_requests{route}
order_service_dependency_requests_total{service,operation,result}
order_service_dependency_duration_seconds{service,operation}
order_service_queue_depth{queue}
order_service_orders_total{result}
```

## 阶段交付物

- 一份指标设计表和禁止 label 列表。
- 可重复注册、可单元测试的 metrics package。
- HTTP middleware 和依赖客户端埋点。
- 一次高基数模拟实验，包含序列数量变化和修复结论。
- `/metrics` 的访问控制方案。

## 阶段验收

发起成功、400、500 和超时请求后，你能用 `curl` 和 PromQL 对上：

- 每类请求数量。
- 错误率。
- P95 延迟。
- 当前并发请求。
- 依赖错误和依赖延迟。
- 业务订单成功或失败。

如果这些数字对不上，先修复埋点语义，再进入 PromQL 阶段。

