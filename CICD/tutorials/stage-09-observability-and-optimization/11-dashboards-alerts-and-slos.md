# 11：Dashboard、告警与 SLO

## 1. 本节目标

有了指标，还要让它们变成可读的 dashboard 和有用的告警。

这一节学习：

- 如何设计 CI/CD dashboard。
- 如何设计发布后观察 dashboard。
- 如何写基础告警规则。
- 如何理解 SLO 和错误预算。
- 如何避免告警噪音。

## 2. Dashboard 不是越多越好

好的 dashboard 回答问题。

坏的 dashboard 堆满图。

CI/CD dashboard 建议回答：

```text
PR 反馈是否变慢？
失败率是否升高？
最常失败的 job 是什么？
部署是否成功？
发布后服务是否健康？
```

应用 dashboard 建议回答：

```text
请求量是否正常？
错误率是否异常？
延迟是否变差？
资源是否接近饱和？
最近发布是否相关？
```

## 3. CI/CD Dashboard 面板

建议面板：

| 面板 | 含义 |
| --- | --- |
| PR CI p50/p95 duration | 开发者等待时间 |
| Workflow success rate | 流水线健康度 |
| Failure by job | 失败热点 |
| Queue time | runner 是否不足 |
| Cache hit rate | 缓存是否有效 |
| Image build duration | 构建瓶颈 |
| Deployment duration | 发布速度 |
| Smoke test result | 发布后验证 |

如果暂时没有 Grafana，可以先用 Markdown 周报或表格记录。

## 4. 发布观察 Dashboard

production 发布后重点看：

- 当前版本。
- 最近部署事件。
- 请求量。
- 5xx 错误率。
- p95/p99 延迟。
- Pod 重启。
- CPU/内存。
- 数据库错误。
- 关键业务指标。

发布事件最好能在图上标记。

这样你能看到：

```text
错误率升高是否发生在部署之后。
```

## 5. 基础 Prometheus 告警

示例：高错误率。

```yaml
groups:
  - name: go-cicd-lab
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(go_cicd_lab_http_requests_total{status=~"5.."}[5m]))
          /
          sum(rate(go_cicd_lab_http_requests_total[5m]))
          > 0.05
        for: 10m
        labels:
          severity: page
        annotations:
          summary: "High 5xx error rate"
          description: "5xx error rate is above 5% for 10 minutes."
```

示例：高延迟。

```yaml
- alert: HighLatencyP95
  expr: |
    histogram_quantile(
      0.95,
      sum(rate(go_cicd_lab_http_request_duration_seconds_bucket[5m])) by (le)
    ) > 1
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "High p95 latency"
```

## 6. SLO 是什么

SLO 是 Service Level Objective。

例子：

```text
30 天内 99.9% 的 HTTP 请求成功。
30 天内 95% 的请求延迟低于 300ms。
```

错误预算：

```text
如果目标是 99.9%，那么 0.1% 是允许失败预算。
```

CI/CD 与 SLO 的关系：

```text
发布变更不应持续消耗错误预算。
如果发布后错误预算快速燃烧，应触发回滚或冻结发布。
```

## 7. 告警要能行动

差告警：

```text
CPU > 70%
```

不一定有问题。

好告警：

```text
production 5xx 错误率超过 5%，持续 10 分钟，且当前有真实流量。
```

告警应该包含：

- 影响范围。
- 当前值和阈值。
- 可能原因。
- runbook 链接。
- dashboard 链接。
- 最近部署链接。

## 8. 告警降噪

常见噪音：

- 无流量时错误率计算异常。
- 每个实例都单独告警。
- 短暂抖动立刻告警。
- 同一个故障触发十几个告警。

处理：

- 设置 `for` 持续时间。
- 按服务聚合。
- 区分 warning 和 page。
- 低优先级只进 issue 或 chat。
- 告警必须对应 runbook。

## 9. CI/CD 自身告警

可以为流水线设置：

- main 分支 CI 连续失败。
- production deploy 失败。
- image signing 失败。
- Argo CD app Degraded。
- smoke test 失败。
- 变更失败率连续上升。

但不要给每个 PR 失败都发高优先级告警。

PR 失败通常应反馈给 PR 作者，而不是叫醒运维。

## 10. 小练习

完成：

1. 设计一个 CI/CD dashboard 草图。
2. 设计一个 production 发布观察 dashboard 草图。
3. 写一个高错误率告警规则。
4. 写一个 smoke test 失败告警策略。
5. 为每个告警补充 runbook 链接。

## 11. 本节小结

你现在应该理解：

- dashboard 应围绕问题设计。
- 发布观察 dashboard 要把部署事件和运行时指标联系起来。
- 告警必须可行动，不能只是噪音。
- SLO 让你用用户体验约束发布质量。
- CI/CD 自身也需要告警，但优先级要谨慎设计。

