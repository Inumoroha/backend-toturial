# 性能、升级与日常运维

## 1. 查询性能排查

当 Grafana 或 API 查询变慢：

1. 记录完整 PromQL、时间范围和 step。
2. 在 Prometheus Graph 直接执行。
3. 看查询耗时、返回序列数和失败。
4. 把 selector、窗口、聚合一步步拆开。
5. 检查是否使用了高基数 label、子查询或大范围 `rate`。
6. 将反复使用的计算转成 Recording Rule。
7. 再评估内存、磁盘和远程读取。

常见自监控：

```promql
histogram_quantile(
  0.95,
  sum by (le) (rate(prometheus_engine_query_duration_seconds_bucket[5m]))
)
rate(prometheus_engine_query_failures_total[5m])
```

## 2. 抓取性能

检查：

```promql
max by (job) (scrape_duration_seconds)
sum by (job) (scrape_samples_scraped)
sum by (job) (scrape_samples_post_metric_relabeling)
rate(scrape_samples_scraped[5m])
```

如果抓取耗时接近 interval：

- target endpoint 太慢。
- metrics 输出过大。
- DNS 或网络不稳定。
- Prometheus 资源不足。
- 抓取并发和目标数量过高。

不要只把 timeout 调大；这可能减少失败表象但延迟数据新鲜度。

## 3. 高基数治理

建立周期检查：

- 新增指标和 label 的 code review。
- `head_series` 趋势。
- 每个 job 的 samples。
- 最近部署前后的增长。
- Top label value 数量。
- 远程写入样本成本。

发现问题时：

1. 确定产生者。
2. 暂时减少查询和通知压力。
3. 修复应用埋点。
4. 删除旧序列只能等待保留/重启/清理策略，不要依赖简单重命名。
5. 复盘为什么评审没有阻止它。

## 4. 升级流程

1. 阅读当前版本和目标版本的 breaking changes。
2. 在测试环境复制配置和代表性数据量。
3. 运行 `promtool check config` 和规则测试。
4. 验证 scrape、query、rule、remote write、Grafana 和 Alertmanager。
5. 备份配置、规则、Dashboard 和恢复信息。
6. 分批升级，观察自监控和告警。
7. 保留回滚版本和明确的停止条件。
8. 升级后检查数据新鲜度、规则延迟和通知链路。

## 5. 维护 Runbook

至少写这些 Runbook：

- Prometheus 磁盘即将耗尽。
- Prometheus 内存持续升高。
- 查询大量超时。
- 远程写入队列积压。
- 规则评估失败。
- Alertmanager 通知失败。
- Grafana 数据源不可用。
- 发现高基数指标。

## 6. 日常检查清单

```text
[ ] Prometheus ready/healthy
[ ] target down 数量
[ ] 规则评估失败
[ ] 告警通知失败
[ ] remote write pending/failed
[ ] 磁盘和内存余量
[ ] head series 趋势
[ ] 查询耗时 P95
[ ] 最近配置和指标变更
[ ] 是否有未处理 silence
```

## 7. 阶段验收

完成一次“慢查询 + 高基数 + 远程写入失败”的组合演练，记录你如何排序问题、先保护系统再修复根因，并说明哪些指标证明系统已经恢复。

建议把演练拆成可重复的脚本步骤：

1. 保存基线：head series、查询 P95、remote write pending、磁盘和内存。
2. 执行一条无 selector 的大范围查询，记录耗时和返回序列数。
3. 注入无界 label，等待一个 scrape interval 后记录 series 增长。
4. 暂停远程存储，观察失败和积压。
5. 先停止高成本查询，再修复应用 label，最后恢复远程存储。
6. 重新执行基线查询，提供恢复时间和残留数据影响。
