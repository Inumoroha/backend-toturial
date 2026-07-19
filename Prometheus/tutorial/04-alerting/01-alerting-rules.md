# Alerting Rule 设计

## 0. 把规则接入 Prometheus

Prometheus 只有加载了规则文件才会评估其中的 Recording Rule 和 Alerting Rule。主配置至少需要：

```yaml
rule_files:
  - /etc/prometheus/rules/*.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093
```

容器目录要通过 volume 挂载，例如：

```yaml
services:
  prometheus:
    volumes:
      - ./prometheus/rules:/etc/prometheus/rules:ro
  alertmanager:
    image: prom/alertmanager:v0.27.0
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
```

修改后按顺序验证：

```bash
promtool check config prometheus/prometheus.yml
promtool check rules prometheus/rules/*.yml
curl -X POST http://localhost:9090/-/reload
curl http://localhost:9090/api/v1/rules
curl http://localhost:9090/api/v1/alerts
```

在 `/rules` 页面确认规则组的 `Last Evaluation` 和错误信息；在 `/alerts` 页面确认告警状态。不要只看配置文件没有报错就认为告警已经生效。

## 1. 先定义告警对象

常见对象分为三类：

- 用户影响：错误率、SLO 达标率、可用性。
- 服务健康：target down、进程重启、队列积压、依赖超时。
- 监控系统健康：规则评估失败、抓取失败、远程写入积压。

优先告警用户影响。CPU 80% 不是故障本身，只有当它导致延迟、错误或容量风险时才值得升级。

## 2. Target down

```yaml
groups:
  - name: availability
    rules:
      - alert: TargetDown
        expr: up{job=~"go-app|order-worker"} == 0
        for: 5m
        labels:
          severity: page
          team: backend
          service: "{{ $labels.job }}"
          environment: production
        annotations:
          summary: "目标不可抓取"
          description: "job={{ $labels.job }}, instance={{ $labels.instance }} 已连续 5 分钟不可抓取"
          runbook_url: "https://example.invalid/runbooks/target-down"
```

`up == 0` 只能说明 Prometheus 无法成功抓取 target，不等于应用所有业务都失败。把它和业务错误率分成两条告警。

## 3. 错误率告警

```yaml
      - alert: OrderAPIHighErrorRate
        expr: |
          (
            sum by (job) (
              rate(order_service_http_requests_total{status=~"5.."}[5m])
            )
            /
            sum by (job) (
              rate(order_service_http_requests_total[5m])
            )
          ) > 0.05
        for: 10m
        labels:
          severity: page
          team: backend
          service: order-api
          environment: production
        annotations:
          summary: "订单 API 错误率超过 5%"
          description: "job={{ $labels.job }} 当前错误率={{ $value | humanizePercentage }}"
          runbook_url: "https://example.invalid/runbooks/order-api-error-rate"
```

注意：

- 分子和分母必须使用相同的聚合维度。
- 低流量服务可以增加最小请求量条件。
- 错误率告警要有恢复条件，恢复通常由同一表达式低于阈值自然实现。
- 使用 `for` 时要考虑 scrape interval 和实际故障反应时间。

## 4. 延迟告警

```yaml
      - alert: OrderAPIP95TooHigh
        expr: |
          histogram_quantile(
            0.95,
            sum by (le, route) (
              rate(order_service_http_request_duration_seconds_bucket[10m])
            )
          ) > 0.3
        for: 15m
        labels:
          severity: ticket
          team: backend
          service: order-api
          environment: production
        annotations:
          summary: "订单 API P95 超过 300ms"
          description: "route={{ $labels.route }}, p95={{ $value }}s"
          runbook_url: "https://example.invalid/runbooks/order-api-latency"
```

P95 是估计值。规则中必须确认 bucket 足以表达 300ms 附近的差异。

## 5. 队列积压

```yaml
      - alert: OrderQueueBacklog
        expr: order_service_queue_depth{queue="orders"} > 1000
        for: 15m
        labels:
          severity: page
          team: backend
          service: order-worker
          environment: production
        annotations:
          summary: "订单队列持续积压"
          description: "queue={{ $labels.queue }}, depth={{ $value }}"
          runbook_url: "https://example.invalid/runbooks/order-queue-backlog"
```

如果队列深度为 Gauge，判断“持续增长”可以结合处理速率和生产速率，而不是只看绝对值。

## 6. SLO burn rate

先定义：

```text
SLO = 99.9%
error_budget_ratio = 0.001
```

先定义短窗口和长窗口的 Recording Rule，确保告警表达式引用的指标真实存在：

```yaml
groups:
  - name: order-api-slo-recording
    interval: 30s
    rules:
      - record: job:http_error_ratio:5m
        expr: |
          sum by (job) (rate(order_service_http_requests_total{status=~"5.."}[5m]))
          /
          sum by (job) (rate(order_service_http_requests_total[5m]))

      - record: job:http_error_ratio:1h
        expr: |
          sum by (job) (rate(order_service_http_requests_total{status=~"5.."}[1h]))
          /
          sum by (job) (rate(order_service_http_requests_total[1h]))
```

然后为短窗口和长窗口分别记录错误比例，避免短暂尖峰或慢性退化单独触发 page：

```yaml
      - alert: OrderAPIHighErrorBudgetBurn
        expr: |
          (
            job:http_error_ratio:5m{job="order-api"} > 14.4 * 0.001
          )
          and
          (
            job:http_error_ratio:1h{job="order-api"} > 14.4 * 0.001
          )
        for: 2m
        labels:
          severity: page
          team: backend
          service: order-api
          environment: production
        annotations:
          summary: "订单 API 快速消耗错误预算"
          description: "短窗口和长窗口都超过 burn rate 阈值"
          runbook_url: "https://example.invalid/runbooks/slo-burn"

```

系数、窗口和通知级别必须根据 SLO 窗口、业务重要性和响应能力调整。

规则验收时先分别查询两个 Recording Rule，再查询告警表达式；如果其中一个窗口没有样本，应该检查流量、抓取状态和低流量策略，而不是直接用 `or vector(0)` 隐藏问题。

## 7. 告警命名和标签

推荐标签：

```yaml
labels:
  severity: page
  team: backend
  service: order-api
  environment: production
```

把 route、instance 等影响分组的维度保留在告警标签；把当前数值和说明放入 annotations。不要把长错误文本放在 label。

## 8. 告警评审问题

- 用户现在受到什么影响？
- 这个告警需要立即叫人，还是工作时间处理？
- 短暂抖动会不会触发？
- 是否存在同一事故的更高层告警？
- 收到通知的人能否在 5 分钟内开始排查？
- 解决后是否会自动恢复？
- 是否有测试覆盖阈值边界和缺失数据？

## 9. 阶段练习

实现并验证：

1. `TargetDown`。
2. 5xx 错误率。
3. P95 延迟。
4. 队列积压。
5. Prometheus 规则评估失败。
6. 自身 scrape 失败。

为每条写 severity、team、service、Runbook 和 `for`。
