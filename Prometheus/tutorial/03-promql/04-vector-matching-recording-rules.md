# 向量匹配、Recording Rule 与查询优化

## 1. 二元运算怎样匹配

两个 instant vector 做除法时，PromQL 默认按完整 label 集合匹配。下面两个结果可能无法匹配：

```promql
sum by (route) (rate(http_requests_total[5m]))
/
sum by (route, status) (rate(http_requests_total[5m]))
```

需要让两边输出相同的 label 集合，或者明确使用 `on`/`ignoring`：

```promql
sum by (route) (rate(http_requests_total[5m]))
/
on (route)
sum by (route) (rate(http_requests_total{status=~"5.."}[5m]))
```

`on (route)` 表示只使用 route 匹配。写完要检查结果数量，避免无意中匹配多条序列。

## 2. group_left 和 group_right

当一边是一对多关系时使用：

```promql
sum by (instance) (rate(container_cpu_usage_seconds_total[5m]))
/
on (instance) group_left
machine_cpu_cores
```

谨慎使用 `group_left`：

- 先确认哪边是“一”。
- 确认右侧 label 不会导致重复。
- 查询后检查序列数量和样本值。
- 不要用它掩盖 label 设计不一致。

## 3. Recording Rule

适合做 Recording Rule 的查询：

- 多个 Dashboard 重复使用。
- 告警高频评估。
- 计算复杂、数据量大。
- 需要固定规范名称供团队复用。

示例：

```yaml
groups:
  - name: order-service-recording
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      - record: job:http_errors:rate5m
        expr: sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))

      - record: job:http_error_ratio:rate5m
        expr: |
          job:http_errors:rate5m
          /
          job:http_requests:rate5m
```

名称应体现聚合维度和时间窗口。规则输出的 label 要稳定，避免下游查询不断猜测。

## 4. 规则加载和验证

```bash
promtool check rules prometheus/rules/*.yml
```

修改 Prometheus 配置后：

```bash
curl -X POST http://localhost:9090/-/reload
```

到 `/rules` 确认：

- group 是否存在。
- evaluation time 是否更新。
- last evaluation 是否报错。
- recording rule 是否产生序列。

## 5. 查询性能

优化顺序：

1. 缩小时间范围。
2. 用最具体的 selector，避免无约束扫描。
3. 先过滤，再聚合。
4. 只保留必要 label。
5. 把重复复杂表达式预计算。
6. 控制 Grafana refresh 和 panel 数量。
7. 检查远程读取、内存和查询并发。

不要为了“看起来快”随意删除业务维度。先证明该维度不支持决策，再移除。

## 6. 常见错误

- 直接把两个不同单位的指标相除。
- 计算 Histogram 分位数时忘记 `le`。
- 分子和分母 label 不一致。
- 用 `or vector(0)` 掩盖抓取失败。
- 对低流量服务使用过短窗口。
- 使用 `label_replace` 生成高基数值。
- Recording Rule 名称不包含窗口或聚合语义。

## 7. 阶段项目

创建 `rules/recording.yml`，包含：

- 服务请求速率。
- 服务错误率。
- route P95。
- SLO 达标率。
- Worker 处理速率。
- 队列积压。

每条规则附一段注释，说明输入指标、输出 label、评估间隔和使用者。

## 8. 向量匹配实验

先准备两组语义明确的指标：

```text
service_requests_total{service="api",instance="a"}
instance_cpu_cores{instance="a"}
```

依次执行：

```promql
rate(service_requests_total[5m])
rate(service_requests_total[5m]) / instance_cpu_cores
rate(service_requests_total[5m]) / on(instance) instance_cpu_cores
```

记录每一步的结果数量。再人为给 `instance_cpu_cores` 增加一个 `source` label，观察默认匹配和 `on(instance)` 的差别；最后使用 `group_left` 前先确认右侧每个 instance 只有一个值。

## 9. 查询验收

- [ ] 能解释默认完整 label 匹配为什么失败。
- [ ] 能写出 `on`、`ignoring`、`group_left` 的最小例子。
- [ ] 能通过 `count()` 检查 join 后是否意外增加序列。
- [ ] 能把至少三条重复高成本查询转换成 Recording Rule。
- [ ] 能在 `/rules` 页面确认评估时间、错误和输出序列。
