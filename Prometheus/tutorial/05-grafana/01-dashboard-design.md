# Dashboard 设计与查询组织

## 1. 数据源 provisioning

可以用文件预配置 Prometheus 数据源：

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

Dashboard provider 让 Grafana 启动时从 Git 管理的 JSON 目录加载面板：

```yaml
apiVersion: 1

providers:
  - name: prometheus-lab
    orgId: 1
    folder: Prometheus Lab
    type: file
    disableDeletion: true
    allowUiUpdates: false
    options:
      path: /var/lib/grafana/dashboards
```

Compose 挂载方式：

```yaml
services:
  grafana:
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
```

`allowUiUpdates: false` 表示 UI 不是唯一事实来源。开发时可以临时开启 UI 编辑，验证后导出 JSON、清理敏感字段并提交 Git，再恢复代码化加载。

Grafana 容器中的 `prometheus` 是 Compose 服务名。浏览器能访问 Grafana，不代表浏览器应该直接访问 Prometheus；使用 proxy 让 Grafana 服务端访问数据源。

## 2. Dashboard 行布局

### Row 1：SLO 和健康

- 可用性或 SLO 达标率。
- 当前错误预算消耗。
- `sum(up{job=~"$service"})`。
- Firing 告警数量。

### Row 2：RED

```promql
sum by (job) (rate(order_service_http_requests_total{job=~"$service"}[5m]))
```

```promql
sum(rate(order_service_http_requests_total{status=~"5..",job=~"$service"}[5m]))
/
sum(rate(order_service_http_requests_total{job=~"$service"}[5m]))
```

```promql
histogram_quantile(
  0.95,
  sum by (le, route) (
    rate(order_service_http_request_duration_seconds_bucket{job=~"$service"}[5m])
  )
)
```

### Row 3：拆分

- 按 route 的请求速率。
- 按 status 的错误率。
- 按 instance 的 P95。
- Top 5 慢 route。

### Row 4：依赖和资源

- 下游调用 P95。
- 下游错误率和超时。
- 队列深度与消费速率。
- goroutine、GC、连接池。

## 3. 变量

变量示例：

```promql
label_values(up, job)
label_values(up{job=~"$service"}, instance)
label_values(up{job=~"$service"}, environment)
```

变量要设置：

- 是否多选。
- 是否包含 All。
- All 的正则表达式。
- 没有数据时的行为。
- 变量变化后每个 Panel 是否仍然有合法 selector。

## 4. 单位和阈值

- 请求速率：req/s。
- 比率：percent (0-100) 或 ratio (0-1)，全局统一。
- 延迟：seconds 或 milliseconds，避免同一 Dashboard 混用。
- 字节：bytes。
- 队列：short 或 messages。

阈值来自 SLO、容量或历史基线。不要因为图线看起来刺眼就随意设置红色。

## 5. Dashboard as Code

建议保存：

```text
grafana/
├── provisioning/datasources/prometheus.yml
├── provisioning/dashboards/default.yml
└── dashboards/order-service.json
```

导出 JSON 后检查：

- 是否包含不可复现的数据源 UID。
- 是否把密码、token 写入 JSON。
- 是否固定了无意义的 panel ID。
- 查询变量是否与部署环境匹配。

## 6. 慢 Panel 排查

1. 在 Panel Inspector 查看实际请求。
2. 把时间范围缩短到 15 分钟。
3. 单独执行 PromQL，观察返回序列数量。
4. 删除不必要的 label 维度。
5. 使用 Recording Rule。
6. 降低刷新频率或延长最小 step。
7. 检查 Prometheus 的查询耗时和内存。

## 7. 设计练习

为“订单 API 变慢”设计一张一屏 Dashboard，限制在 12 个 Panel 以内。每个 Panel 写明：

- 用户问题。
- PromQL。
- 单位。
- 保留的 label。
- 需要跳转到的 Runbook 或细节页面。

## 8. 阶段验收

在没有教程提示的情况下，完成一次 Dashboard 导出、删除、重新 provisioning，并确认查询和变量行为不变。
