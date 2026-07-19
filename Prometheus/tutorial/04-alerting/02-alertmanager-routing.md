# Alertmanager 路由与抑制

## 1. Alertmanager 解决什么

Prometheus 决定“是否有告警”，Alertmanager 决定：

- 哪些告警属于同一组。
- 发给哪个团队或渠道。
- 什么时候首次发送和重复发送。
- 哪些告警可以被静默。
- 哪些低级告警应该被更高层告警抑制。

不要把通知路由逻辑全部写进 PromQL；规则负责准确表达状态，Alertmanager 负责通知策略。

## 2. 最小配置

```yaml
global:
  resolve_timeout: 5m

route:
  receiver: default-webhook
  group_by: ["alertname", "service", "environment"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - matchers:
        - severity="page"
    - matchers:
        - severity="ticket"
      receiver: ticket-webhook
      group_wait: 10m

receivers:
  - name: default-webhook
    webhook_configs:
      - url: "http://notification-adapter:8080/alerts"

  - name: ticket-webhook
    webhook_configs:
      - url: "http://ticket-adapter:8080/alerts"

inhibit_rules:
  - source_matchers:
      - alertname="TargetDown"
      - severity="page"
    target_matchers:
      - severity="ticket"
    equal: ["environment", "service"]
```

配置中的 URL 是实验地址，生产应使用 Secret、TLS、认证和可审计的通知适配器。

## 3. 分组策略

`group_by` 过少：

- 不同服务的故障合并在一条消息里。
- 值班人无法知道影响范围。

`group_by` 过多：

- 一次事故产生大量独立通知。
- 相同根因被拆散。

通常按 `alertname`、`service`、`environment` 开始，再根据告警风暴实验调整。

## 4. group_wait 和 repeat_interval

- `group_wait`：新组产生后等待多久再发第一条，给同一故障的相关告警聚合时间。
- `group_interval`：已发送过的组新增告警后，多久再发送更新。
- `repeat_interval`：告警持续未恢复时的重复提醒间隔。

不要把 repeat interval 设成几分钟就用于生产，否则值班频道会被刷屏。

## 5. 静默 Silence

静默适合：

- 已批准的维护窗口。
- 正在进行的发布。
- 已知且有人负责的短期故障。

静默必须包含创建人、原因、起止时间和关联变更。不要用永久静默隐藏噪音；噪音应该修复规则、服务或路由。

## 6. 抑制 Inhibition

典型关系：

```text
集群不可达
  -> 抑制节点资源告警
服务完全不可抓取
  -> 抑制服务内部延迟告警
```

抑制条件必须足够精确。过宽的 `equal` 可能把不相关告警一起隐藏。

## 7. 可靠性和安全

- Alertmanager 自身要有监控和健康检查。
- Webhook 要有超时、重试和认证。
- 通知内容不要包含 token、密码和敏感请求字段。
- 多副本 Alertmanager 需要按官方方式配置集群，不要只启动两个独立实例。
- 通知失败要产生可观测指标并进入排障流程。

## 8. 练习

1. 创建 page 和 ticket 两条路由。
2. 触发同一服务的 10 条告警，观察分组。
3. 创建一个 10 分钟静默，确认通知停止。
4. 触发 TargetDown 和下游延迟，验证 inhibit。
5. 恢复服务，确认 Resolved 通知是否符合渠道约定。

配置变更后至少执行：

```bash
amtool check-config alertmanager/alertmanager.yml
curl http://localhost:9093/-/ready
curl http://localhost:9093/api/v2/status
```

如果环境没有 `amtool`，使用与 Alertmanager 镜像相同版本的二进制或容器执行校验。配置通过校验不代表通知一定能送达，还要在演练中验证 receiver 的网络、认证、超时和重试。

## 9. 验收

给同学一个包含 3 个服务和 20 条告警的事件，他能从通知判断：

- 根因候选。
- 受影响服务。
- 当前严重程度。
- 该看哪个 Runbook。
- 哪些告警只是被抑制而不是已经解决。
