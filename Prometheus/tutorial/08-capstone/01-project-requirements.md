# 项目需求、架构与指标契约

## 1. 业务流程

### 创建订单

1. 客户端调用 `POST /orders`。
2. Order API 校验商品和数量。
3. 调用 Inventory 检查库存。
4. 调用 Payment 创建支付。
5. 写入订单状态。
6. 发布订单事件给 Worker。
7. 返回订单 ID 和状态。

### 异步处理

Worker 从队列读取订单事件：

- 处理成功：记录完成。
- 依赖失败：按策略重试。
- 超过重试次数：进入 dead-letter 或失败状态。
- 队列持续没有消费者：触发积压告警。

## 2. 服务边界

| 服务 | 端口 | 职责 | 指标重点 |
| --- | ---: | --- | --- |
| order-api | 8080 | HTTP API | RED、业务结果、依赖 |
| order-worker | 8081 | 消费队列 | 消费速率、处理延迟、积压 |
| fake-dependency | 8082 | 故障可控的库存/支付 | 延迟、错误、请求量 |

metrics 可以使用独立端口 9091、9092、9093，避免业务端点和管理端点混在一起。

## 3. 指标契约

### HTTP

```text
order_service_http_requests_total{method,route,status}
order_service_http_request_duration_seconds{method,route}
order_service_http_in_flight_requests{route}
```

### 依赖

```text
order_service_dependency_requests_total{service,operation,result}
order_service_dependency_duration_seconds{service,operation}
```

### Worker

```text
order_worker_messages_total{result}
order_worker_processing_duration_seconds
order_worker_queue_depth{queue}
order_worker_retries_total{reason}
```

### 业务

```text
order_service_orders_total{result}
order_service_payment_total{result}
order_service_inventory_check_total{result}
```

所有 label 值域写入 `metric-design.md`，包括 route、status、result、service 和 operation 的允许值。

## 4. SLO 示例

先定义清楚后再写告警：

```text
订单 API 可用性：30 天内成功请求比例 >= 99.9%
订单 API 延迟：纳入 SLO 的请求中 99% <= 500ms
Worker：15 分钟内队列深度不超过 1000，且处理速率 >= 生产速率
```

说明：

- 哪些 route 纳入 SLO。
- 4xx 是否算用户错误。
- health 和 metrics 是否排除。
- 低流量时如何避免比率噪音。
- 依赖失败由哪个服务负责告警。

## 5. Compose 启动阶段

第一版只启动：

```text
order-api
order-worker
fake-dependency
prometheus
grafana
alertmanager
```

先用内存队列，确保监控链路完成。第二版再把队列替换为 Redis/NATS/Kafka，并保留同一套业务指标语义。

## 6. 实现顺序

1. 业务接口和状态码。
2. 单元测试和故障注入配置。
3. metrics package 和 `/metrics`。
4. Prometheus target。
5. RED 和依赖 Dashboard。
6. Recording/Alerting Rules。
7. Alertmanager 路由。
8. Worker 和队列指标。
9. K8s 部署。
10. 容量与 HA 文档。

## 7. 代码评审标准

- 每个指标有 Help、单位和设计记录。
- middleware 不记录原始 URL。
- 依赖重试和用户请求的计数边界清楚。
- metrics handler 有超时和访问控制。
- 配置中的秘密通过环境变量或 Secret 注入。
- 故障注入默认关闭。

