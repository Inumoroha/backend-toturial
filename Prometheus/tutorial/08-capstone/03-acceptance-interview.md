# 验收、复盘与面试表达

## 1. 自动化验收命令

建议在仓库根目录提供：

```bash
go test ./...
go test -race ./...
go vet ./...
promtool check config prometheus/prometheus.yml
promtool check rules prometheus/rules/*.yml
promtool test rules prometheus/tests/*.yml
docker compose config
docker compose up -d
```

Kubernetes 环境再加入：

```bash
kubectl apply --dry-run=client -f deploy/k8s/
kubectl get pods,svc -n orders
kubectl get servicemonitor -n orders
```

版本差异可能改变命令参数，脚本中应先检查工具可用性并输出版本。

## 2. 功能验收矩阵

| 能力 | 验收证据 |
| --- | --- |
| API | 测试覆盖成功、4xx、5xx、超时 |
| 埋点 | `/metrics` 包含预期名称和 labels |
| 抓取 | targets 全部 UP，故障时错误可解释 |
| PromQL | 30 条查询笔记和实际结果 |
| Histogram | P95/P99 和 SLO 达标率 |
| 规则 | 静态检查和 rule unit tests |
| 通知 | Pending/Firing/Resolved 样例 |
| Dashboard | 四张可复现 Dashboard |
| K8s | ServiceMonitor 和 RBAC 证据 |
| 生产 | 容量、HA、安全和恢复文档 |
| 故障 | 三次演练复盘 |

## 3. 作品集 README 结构

```text
项目简介
架构图
快速启动
版本与前置条件
服务和端口
指标契约
PromQL 示例
Dashboard
告警与 Runbook
故障注入
容量估算
安全边界
已知限制
升级和恢复
事故复盘
```

## 4. 面试场景表达

### 场景：接口变慢

按这个顺序回答：

1. Dashboard 先确认影响范围和开始时间。
2. 用请求速率和错误率确认是否伴随失败。
3. 用 route 和 instance 拆分。
4. 用 Histogram P95/P99 观察长尾。
5. 对比依赖调用延迟、连接池和队列。
6. 用 CPU、内存、GC、goroutine 排除资源瓶颈。
7. 用日志/Tracing 定位单请求。
8. 采取降级、限流、回滚或扩容。
9. 用告警和恢复指标验证修复。
10. 补齐缺失的观测和测试。

### 场景：Prometheus 抓不到服务

回答必须包含：

- `/targets` 的 Last Error。
- 从 Prometheus 网络内 curl `/metrics`。
- DNS、端口、路径、HTTP 状态和格式。
- Kubernetes selector、端口名、RBAC 和 NetworkPolicy。
- 修复后的恢复证据。

### 场景：序列突然暴涨

回答必须包含：

- `head_series` 和 samples 趋势。
- 最近部署和新增 label。
- URL/ID/错误文本等高基数来源。
- 先保护 Prometheus，再修复应用。
- 如何通过 code review、测试和预算阻止复发。

## 5. 最终自评

每项 0～2 分：

- 0：无法完成。
- 1：能照文档完成。
- 2：能解释原理并处理变体。

```text
Go 服务和测试：
网络排障：
指标设计：
Go 埋点：
PromQL：
Histogram/SLO：
告警：
Alertmanager：
Grafana：
Exporter：
Kubernetes：
TSDB/容量：
HA/远程存储：
安全：
故障复盘：
```

总分低于 20 分时，回到对应阶段重新做实验；不要只靠背面试答案。

## 6. 交付证据包

最终仓库应能直接找到这些文件：

```text
docs/architecture.md
docs/metric-design.md
docs/capacity.md
prometheus/rules/recording.yml
prometheus/rules/alerts.yml
prometheus/tests/alerts.yml
grafana/dashboards/order-service.json
runbooks/OrderAPIHighErrorRate.md
runbooks/TargetDown.md
postmortems/*.md
```

每个证据文件都要包含版本、执行日期和实验环境。截图只能作为辅助证据，关键查询、规则和配置必须以文本形式纳入 Git。
