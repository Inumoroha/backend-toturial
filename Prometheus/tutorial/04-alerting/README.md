# 阶段 4：告警与 Alertmanager

指标只有在能触发正确行动时才真正有价值。本阶段把 PromQL 变成告警规则，再把告警变成不会淹没值班人员的通知。

## 学习目标

- 区分 symptom、cause 和 informational alert。
- 编写带 `for`、严重级别、责任团队和 Runbook 的规则。
- 处理 target down、错误率、延迟、资源和 SLO burn rate。
- 使用 `promtool` 测试规则。
- 配置 Alertmanager 分组、路由、静默和抑制。
- 验证 Pending、Firing、Resolved 和重复通知行为。

## 教程顺序

1. [Alerting Rule 设计](01-alerting-rules.md)
2. [Alertmanager 路由与抑制](02-alertmanager-routing.md)
3. [规则测试与 Runbook](03-rule-tests-runbooks.md)

## 一条告警的生命周期

```text
PromQL expression
  -> rule evaluation
  -> inactive / pending / firing
  -> Alertmanager grouping and routing
  -> receiver notification
  -> silence / inhibit / resolve
```

## 告警设计的最小字段

每条面向人的告警至少包含：

- `alertname`：稳定、可搜索的名字。
- `severity`：例如 `page`、`ticket`、`info`。
- `service`、`team`、`env`：用于路由。
- `summary`：一句话说明症状。
- `description`：影响、时间窗口和当前值。
- `runbook_url`：下一步操作。
- `for`：避免瞬时噪音。

## 阶段交付物

- Recording Rules 和 Alerting Rules。
- `promtool check rules`、`promtool test rules` 通过。
- Alertmanager 配置和通知样例。
- 至少三条 Runbook。
- 一次告警风暴和一次告警恢复演练记录。

