# 规则测试与 Runbook

## 1. 为什么规则必须测试

告警表达式经常在这些场景失效：

- label 聚合不一致。
- Counter reset。
- 没有流量导致除零或空结果。
- `for` 时间理解错误。
- Histogram bucket 不存在。
- target down 后表达式反而变成空值。
- 规则名称或 label 变更导致 Alertmanager 路由失效。

规则测试要把输入序列、评估时间和期望状态固定下来。

## 2. promtool 检查

```bash
promtool check config prometheus/prometheus.yml
promtool check rules prometheus/rules/*.yml
promtool check metrics < metrics.txt
promtool test rules prometheus/tests/*.yml
```

不同版本的 promtool 参数可能变化，先执行 `promtool --help` 确认。

## 3. 规则测试文件

示例：

```yaml
rule_files:
  - ../rules/alerts.yml

evaluation_interval: 1m

tests:
  - interval: 1m
    input_series:
      - series: 'order_service_http_requests_total{job="order-api",status="200"}'
        values: '0 60 120 180 240 300 360 420 480 540 600 660 720 780 840 900'
      - series: 'order_service_http_requests_total{job="order-api",status="500"}'
        values: '0 0 10 20 30 40 50 60 70 80 90 100 110 120 130 140'
    alert_rule_test:
      - eval_time: 15m
        alertname: OrderAPIHighErrorRate
        exp_alerts:
          - exp_labels:
              job: order-api
              severity: page
              team: backend
              service: order-api
              environment: production
```

测试文件中的时间序列值只是示意，必须根据实际 rule 的窗口、阈值和 `for` 调整。不要只测试一个“肯定触发”的场景。

## 4. 必测场景

每条重要告警至少覆盖：

- 正常值不会触发。
- 达到阈值但未持续足够时间仍是 Pending 或不触发。
- 持续超过 `for` 后 Firing。
- 指标恢复后 Resolved。
- 没有流量。
- target down。
- label 维度有多个实例。
- Counter reset。
- 规则表达式变更后的兼容性。

## 5. Runbook 模板

每条 page 告警对应一份 Runbook：

```text
标题：
告警名：
影响：
触发条件：
第一步检查：
第二步检查：
常见根因：
临时缓解：
永久修复：
回滚：
恢复验证：
升级联系人：
相关 Dashboard：
相关日志/Tracing：
最近一次复盘：
```

Runbook 的第一步必须是可执行命令或链接，而不是“联系开发”。

## 6. 告警演练记录

记录一次完整事件：

```text
时间线：
- 10:00 开始注入 5xx
- 10:10 告警进入 Pending
- 10:20 收到 page
- 10:23 定位到支付依赖
- 10:27 降级支付路径
- 10:35 错误率恢复
- 10:39 收到 Resolved

做得好的地方：
问题：
缺失的指标：
误报/漏报：
下次改进：
```

## 7. 阶段验收

- 所有规则可通过静态检查。
- 至少 3 条规则有单元测试。
- 至少 3 份 Runbook 能让陌生工程师开始排查。
- 完成一次告警风暴演练，并说明如何减少通知数量。
