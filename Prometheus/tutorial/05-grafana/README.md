# 阶段 5：Grafana Dashboard

Grafana 的目标是帮助人快速判断影响范围和下一步，而不是把所有指标堆在一张页面上。

## 学习目标

- 配置 Prometheus 数据源。
- 用变量、Panel、阈值和单位设计 Dashboard。
- 按“总览 -> 服务 -> 依赖 -> 资源”组织排障路径。
- 使用 Recording Rule 降低重复查询成本。
- 通过 provisioning 或 JSON 将 Dashboard 纳入 Git。

## 教程顺序

1. [Dashboard 设计与查询组织](01-dashboard-design.md)

## 推荐的四张 Dashboard

1. **平台总览**：SLO、错误预算、服务健康、告警数量。
2. **服务详情**：请求量、错误率、P50/P95/P99、route、status。
3. **依赖与异步任务**：下游延迟、错误、重试、队列深度、消费速率。
4. **资源与 Prometheus 自监控**：CPU、内存、GC、抓取、规则评估和查询。

## 面板设计检查

每个 Panel 都要能回答：

- 指标单位是什么？
- 时间范围和刷新频率是否合理？
- 没有数据与零值如何区分？
- 图例是否包含必要的实例或 route？
- 阈值是否来自 SLO 或容量预算？
- 点击后能否跳到更细的 Dashboard 或 Runbook？

## 阶段交付物

- 一张平台总览 Dashboard。
- 一张服务详情 Dashboard。
- 至少一个变量选择 environment、service、instance。
- 一个使用 Recording Rule 的高频 Panel。
- JSON/provisioning 文件和 README 导入说明。

## 验收

给同学一个“订单 API P95 变高”的场景，他可以只依靠 Dashboard 在 5 分钟内回答：影响 route、开始时间、是否是下游问题、是否集中在某个实例、现在是否恢复。

