# Go 后端工程师的 Prometheus 系统学习路线

> 目标：用一条可执行、可验收的路线，掌握 Prometheus 的使用、开发、运维和故障排查，并能把它应用到 Go 后端服务和 Kubernetes 生产环境中。
>
> 建议投入：每周 8～12 小时，连续 20～24 周。每周都要有代码、查询、图表或故障复盘产出；只看视频不算完成。
>
> 版本说明：Prometheus、Grafana、Go client 的配置项和 UI 会随版本变化。命令和 API 以你实际使用的稳定版本官方文档为准，本文强调不随版本快速变化的概念和工作方法。

## 配套详细教程

本文件负责路线总览、周次安排和能力验收；逐章实操教程位于 [教程总导航](README.md)。教程按阶段拆分为多个目录，包含 Go 服务、Docker Compose、PromQL、告警、Grafana、Kubernetes、生产运维和综合项目。

## 一、完成路线后的能力画像

完成全部阶段后，你应该能够：

- 使用 Docker Compose 或二进制部署 Prometheus、Grafana、Alertmanager 和常见 Exporter。
- 解释 Prometheus 的数据模型、TSDB、抓取模型、Pull 与 Push 的适用边界。
- 为 Go HTTP/RPC 服务设计合理的指标，包括请求量、错误率、延迟、资源使用量和业务指标。
- 熟练编写 PromQL：过滤、聚合、时间函数、同比/环比、直方图分位数、向量匹配、Recording Rule。
- 设计不容易失控的 label，能估算并治理时间序列基数（cardinality）。
- 用 Alerting Rule 和 Alertmanager 实现告警分组、抑制、静默、路由和通知。
- 用 Grafana 构建面向排障的 Dashboard，而不是只展示好看的折线图。
- 在 Kubernetes 中使用 Service Discovery、ServiceMonitor/PodMonitor，并处理权限、网络和标签问题。
- 解释数据保留、远程写入、联邦、HA、长期存储的取舍。
- 通过 Prometheus 自监控、日志、抓取状态和查询性能定位问题。
- 完成一个包含 Go 服务、指标、Dashboard、告警、故障注入和运行手册的可演示项目。

## 二、学习方法和环境

### 2.1 正确顺序

不要把 Prometheus 当成“安装一个监控软件”。对后端工程师而言，最有效的顺序是：

```text
Go 服务基础
  -> 可观测性与指标语义
  -> Prometheus 数据模型和抓取
  -> Go 埋点与 Exporter
  -> PromQL
  -> 告警与 Dashboard
  -> Kubernetes 与服务发现
  -> 存储、HA、性能和生产治理
  -> 综合项目与故障排查
```

每个阶段都遵循：

1. 学概念：能用自己的话解释“为什么”。
2. 做实验：用最小可运行环境验证“怎么做”。
3. 写记录：保存配置、查询、截图和故障结论。
4. 做验收：不看笔记完成阶段任务。

### 2.2 推荐工具

| 工具 | 用途 |
| --- | --- |
| Go 当前稳定版本 | 编写被监控服务、Exporter 和测试 |
| Docker Desktop + Compose | 快速启动 Prometheus、Grafana、Alertmanager |
| Git | 管理实验和配置变更 |
| curl 或 HTTPie | 验证 `/metrics`、Prometheus API 和业务接口 |
| VS Code/GoLand | Go 开发、YAML 编辑 |
| Linux shell 或 WSL | 熟悉生产环境中的日志、进程和网络命令 |
| Grafana | 查询、Dashboard 和告警可视化 |

验证环境：

```bash
go version
docker version
docker compose version
git --version
curl --version
```

Windows 用户可以使用 PowerShell、WSL 或 Git Bash；生产命令最终要能在 Linux 环境中复现。

### 2.3 建议实验仓库结构

从第一天就把实验当成真实工程管理：

```text
prometheus-lab/
├── app/                    # Go 示例服务
│   ├── cmd/server/main.go
│   ├── internal/metrics/
│   └── go.mod
├── prometheus/
│   ├── prometheus.yml
│   ├── rules/
│   └── alerts/
├── alertmanager/
│   └── alertmanager.yml
├── grafana/
│   ├── provisioning/
│   └── dashboards/
├── docker-compose.yml
├── k8s/
├── runbooks/               # 每条告警的排障手册
└── README.md               # 每周实验记录和启动方式
```

## 三、20～24 周分阶段路线

| 阶段 | 周数 | 核心主题 | 阶段产出 |
| --- | ---: | --- | --- |
| 0 | 1～2 | Go 后端与 Linux 基础 | 可测试的 HTTP 服务、并发和性能基线 |
| 1 | 3 | 可观测性与指标设计 | 一份服务监控指标设计文档 |
| 2 | 4～5 | Prometheus 入门与运行 | 本地完整监控栈和抓取实验 |
| 3 | 6～7 | Prometheus 数据模型与 Go 埋点 | 带 RED 指标的 Go 服务 |
| 4 | 8～10 | PromQL 系统训练 | 30 条可复用查询和查询笔记 |
| 5 | 11～12 | Histogram、Summary 与业务指标 | 延迟分位数 Dashboard 和指标规范 |
| 6 | 13～14 | Alerting Rule 与 Alertmanager | 可触发、可路由、带 Runbook 的告警 |
| 7 | 15 | Grafana Dashboard 与可视化 | 面向排障的服务/主机 Dashboard |
| 8 | 16～18 | Exporter、服务发现与 Kubernetes | K8s 中自动发现并监控 Go 服务 |
| 9 | 19～20 | TSDB、远程存储、HA 与性能 | 保留和容量方案、查询优化报告 |
| 10 | 21～24 | 综合项目与面试/生产演练 | 可运行作品集、故障复盘和技术文档 |

时间紧张时，至少完成第 0～7 阶段和综合项目；不要跳过 PromQL、label 设计和告警，因为它们决定实际工作能力。

---

## 阶段 0：Go 后端与 Linux 基础（第 1～2 周）

### 学习目标

Prometheus 只是观测系统。先确保你能读懂被监控服务的请求链路、并发模型、错误处理和资源消耗。

### 必学内容

- Go：module、包、接口、错误包装、context、goroutine、channel、mutex、`defer`。
- HTTP：方法、状态码、超时、连接复用、middleware、JSON、请求取消。
- 服务端：优雅退出、配置加载、日志、健康检查、依赖注入。
- 测试：`testing`、表驱动测试、httptest、race detector、benchmark、pprof 基础。
- Linux：进程、文件描述符、CPU/内存/磁盘/网络、端口、信号、日志。
- 基础网络：DNS、TCP、TLS、反向代理、服务发现。

### 动手任务

1. 写一个 `GET /hello`、`POST /orders`、`GET /healthz` 的 Go HTTP 服务。
2. 增加内存仓库和一个可配置的“随机延迟/随机错误”模拟依赖。
3. 增加优雅退出和请求超时；用 `go test -race ./...` 检查竞态。
4. 用 `go test -bench . -benchmem ./...` 建立基线。
5. 使用 `net/http/pprof` 暴露调试端点，并写下何时开启、如何保护。

### 验收标准

- 能解释一个请求从进入 HTTP server 到返回响应的主要路径。
- 能区分业务错误、客户端错误和服务端错误，并保持状态码语义稳定。
- 能通过日志和 pprof 判断“CPU 高”“内存涨”“请求卡住”的第一步检查方向。

## 阶段 1：可观测性与指标设计（第 3 周）

### 核心概念

- Logs 记录离散事件，Traces 描述单次请求链路，Metrics 描述可聚合的时间序列。
- 先从用户和 SLO 出发：可用性、延迟、吞吐、错误预算，再决定指标。
- RED：Rate（请求速率）、Errors（错误速率）、Duration（延迟）。
- USE：Utilization（利用率）、Saturation（饱和度）、Errors（错误）。
- 指标必须有明确的单位、类型、含义、更新时机和告警用途。

### 指标设计练习

为阶段 0 的订单服务写一份指标表：

| 指标名（示例） | 类型 | Label | 单位 | 用途 |
| --- | --- | --- | --- | --- |
| `http_requests_total` | Counter | `method`, `route`, `status` | 次 | 吞吐和错误率 |
| `http_request_duration_seconds` | Histogram | `method`, `route` | 秒 | 延迟分布和 P95/P99 |
| `http_in_flight_requests` | Gauge | `route` | 个 | 当前并发请求 |
| `order_create_total` | Counter | `result` | 次 | 业务成功/失败趋势 |
| `dependency_request_duration_seconds` | Histogram | `service`, `operation` | 秒 | 下游依赖排障 |

注意：`user_id`、`order_id`、完整 URL、错误堆栈、搜索词等高基数或无界值通常不应该作为 Prometheus label。

### 验收标准

- 能为一个新接口选择 Counter、Gauge、Histogram 或 Summary，并说明理由。
- 能估算某个 label 组合的最大时间序列数量。
- 能从 SLO 推导至少一条可执行告警，而不是只监控“CPU 超过 80%”。

## 阶段 2：Prometheus 入门与运行（第 4～5 周）

### 2.1 先理解架构

```text
被监控服务/Exporter --HTTP /metrics--> Prometheus --PromQL--> Grafana
                                      |
                                      +--> Alerting Rules --> Alertmanager --> 通知渠道
```

重点理解：Prometheus 定期主动抓取目标；目标暴露的是当前可抓取的指标文本；Prometheus 将样本写入本地 TSDB；规则评估和查询都在 Prometheus 侧完成。

### 2.2 最小启动配置

创建 `prometheus/prometheus.yml`：

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: go-app
    metrics_path: /metrics
    static_configs:
      - targets: ["host.docker.internal:8080"]
```

启动 Prometheus：

```bash
docker run --rm -p 9090:9090 \
  -v "$(pwd)/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml" \
  prom/prometheus
```

Windows PowerShell 可将 `$(pwd)` 替换为 `${PWD}`；容器访问宿主机服务时，优先使用 `host.docker.internal`，Linux 环境需按实际网络配置处理。

### 2.3 必须掌握的页面和 API

- `http://localhost:9090/targets`：抓取目标、健康状态、最后错误。
- `http://localhost:9090/graph`：执行 PromQL。
- `http://localhost:9090/rules`：规则加载和评估状态。
- `http://localhost:9090/alerts`：当前告警状态。
- `/-/healthy`、`/-/ready`：Prometheus 自身健康检查。
- `/api/v1/query`、`/api/v1/query_range`：程序化查询。

### 动手任务

1. 手工让一个 target 端口不可用，观察 `up`、`scrape_duration_seconds` 和 `scrape_samples_scraped`。
2. 修改 `scrape_interval`，观察数据点间隔变化。
3. 重启 Prometheus，确认本地数据保留和配置加载行为。
4. 用 API 查询 `up`，把 JSON 结果保存为实验记录。

### 验收标准

- 能判断 target 不健康是网络失败、HTTP 状态异常、响应格式错误还是超时。
- 能解释 `up == 0` 与“应用返回 500”之间的区别。
- 能独立写出一个静态 target、一个带路径的 target 和一个带 `relabel_configs` 的配置。

## 阶段 3：Prometheus 数据模型与 Go 埋点（第 6～7 周）

### 3.1 数据模型

一个时间序列由指标名和完整 label 集合唯一确定：

```text
http_requests_total{method="GET",route="/orders",status="200"}
```

每个样本至少包含时间戳和值。以下规则要形成习惯：

- Counter 只增不减，重启后可归零；使用 `rate`/`increase` 解释它。
- Gauge 可升可降，表示瞬时状态。
- Histogram 产生 `_bucket`、`_sum`、`_count`，适合聚合计算分位数。
- Summary 在客户端计算分位数，跨实例聚合有局限。
- label 名称固定、值域有限、含义稳定；不要把用户输入直接放入 label。
- Counter 名称以 `_total` 结尾；时间以 `_seconds`，字节以 `_bytes`。

### 3.2 Go client 基础实现

依赖：

```bash
go mod init example.com/order-service
go get github.com/prometheus/client_golang
```

最小指标端点：

```go
package main

import (
    "net/http"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var requests = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Namespace: "order_service",
        Subsystem: "http",
        Name:      "requests_total",
        Help:      "Total HTTP requests handled.",
    },
    []string{"method", "route", "status"},
)

func init() {
    prometheus.MustRegister(requests)
}

func main() {
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":8080", nil)
}
```

生产代码不要直接使用 `http.ListenAndServe` 忽略错误；这里仅展示指标注册的最小形态。实际服务应使用自定义 `Registry`、优雅退出和统一 middleware。

### 3.3 HTTP middleware 设计

建议在 middleware 中完成：

1. 记录开始时间。
2. 使用路由模板而不是原始 URL 作为 `route`。
3. 捕获最终状态码和响应时长。
4. 发生 panic 时先恢复并计入错误，再交给统一恢复逻辑。
5. 确保每个请求恰好观测一次。

避免这些常见错误：

- 用 `/orders/12345` 作为 label，导致每个订单创建新序列。
- 将 `error.Error()` 作为 label，造成无界值。
- 在请求热路径中频繁创建 Registry 或动态注册指标。
- 使用 `Summary` 后试图简单地把多个实例的分位数相加或求平均。

### 3.4 自定义 Collector 与进程指标

学习 `prometheus.Collector`、`Describe`、`Collect` 和 `prometheus.NewPedanticRegistry()`。理解默认 Go 进程指标（goroutine、GC、内存、文件描述符）与自定义业务指标的边界。需要时使用 `promhttp.HandlerFor(registry, ...)` 暴露指定 Registry。

### 动手任务

- 给订单服务增加 RED 指标、下游依赖指标、业务成功/失败指标。
- 为 `/metrics` 增加独立端口或网络访问控制。
- 写单元测试检查指标名、label 数量、请求成功/失败计数和 Histogram 样本。
- 用 `promtool check metrics` 检查抓取文本，用 `promtool check config` 检查配置。

### 验收标准

- `curl http://localhost:8080/metrics` 能看到有效指标。
- 发送 100 个请求后，能用 PromQL 对上请求数、错误数和延迟趋势。
- 能解释为什么服务重启后 Counter 归零但 `rate` 不会因此产生错误结论。

## 阶段 4：PromQL 系统训练（第 8～10 周）

PromQL 是这条路线的核心。建议每天至少写 5 条查询，并保存“问题、查询、结果、解释”。

### 4.1 基础选择器与时间范围

```promql
up
up{job="go-app"}
http_requests_total{route="/orders"}
http_requests_total{status=~"5.."}
http_requests_total{route!="/healthz"}
http_request_duration_seconds[5m]
```

理解 instant vector、range vector、标量、字符串，以及 `=~`、`!~` 的正则匹配。

### 4.2 Counter、速率和聚合

```promql
rate(http_requests_total[5m])
sum by (route) (rate(http_requests_total[5m]))
sum by (status) (increase(http_requests_total[1h]))
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

上面的最后两个表达式组合成错误率时，要明确分子和分母的 label 维度；不要把不同单位直接相加或相除。

### 4.3 Gauge、聚合和缺失数据

```promql
max by (instance) (go_goroutines)
avg_over_time(order_queue_depth[15m])
max_over_time(order_queue_depth[1h])
absent(up{job="go-app"})
```

区分“值为 0”“没有样本”“抓取失败”。可用 `or vector(0)` 处理特定展示场景，但告警前必须确认这不会掩盖真实缺失。

### 4.4 Histogram 与延迟分位数

```promql
histogram_quantile(
  0.95,
  sum by (le, route) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

必须理解：`le` 是桶上界；计算分位数前通常先按 `le` 和需要保留的维度聚合；分位数是估计值，受桶边界影响；SLO 需要提前设计合适的 bucket。

### 4.5 向量匹配与多指标关联

```promql
sum by (instance) (rate(container_cpu_usage_seconds_total[5m]))
/
on (instance) group_left
machine_cpu_cores
```

掌握 `on`、`ignoring`、`group_left`、`group_right`，并确认匹配后的时间序列不会意外爆炸。先在小数据集上验证，再用于 Dashboard 或规则。

### 4.6 常用进阶能力

- `sum_over_time`、`avg_over_time`、`min_over_time`、`max_over_time`。
- `delta`、`deriv`、`predict_linear`（适合趋势分析，不能代替容量评审）。
- `offset` 与时间对比。
- 子查询：`max_over_time(rate(...)[1h:5m])`。
- `label_replace`、`label_join`：谨慎用于迁移和兼容。
- `topk`、`bottomk`、`quantile`、`count_values`。
- Recording Rule：将高频、复杂查询预计算，降低 Dashboard 和告警查询压力。

### 4.7 必做的 30 条查询清单

至少完成并解释以下问题：

1. 当前所有 target 的健康状态。
2. 某服务每分钟请求量。
3. 5xx 错误率。
4. 按 route 分组的错误率。
5. P50/P95/P99 延迟。
6. 最近一小时请求总量。
7. 当前 in-flight 请求数。
8. goroutine 数量最高的实例。
9. CPU 使用率最高的实例。
10. 内存工作集最高的实例。
11. 磁盘剩余空间百分比。
12. 过去 24 小时重启次数。
13. 没有抓取到指标的 target。
14. 过去 15 分钟延迟峰值。
15. 与上一小时相比的请求量变化。
16. 按实例汇总的错误率。
17. 失败请求的 route 排名。
18. 各状态码的请求占比。
19. 队列深度的平均值和最大值。
20. 消费者处理速率。
21. 生产速率与消费速率差。
22. 连接池使用率。
23. 下游依赖 P95。
24. 单位时间新增序列数（结合 TSDB 指标）。
25. Prometheus 抓取耗时 P95。
26. Prometheus 查询耗时趋势。
27. 规则评估失败数量。
28. 远程写入失败速率。
29. 服务 SLO 合规率。
30. 多个实例聚合后的全局 P95。

### 验收标准

- 能在不查语法的情况下写出请求速率、错误率和 Histogram P95。
- 能发现并解释错误的 label 匹配、单位不一致和缺失数据。
- 能判断一条查询是否应该变成 Recording Rule，并说明理由。

## 阶段 5：Histogram、Summary 与业务指标（第 11～12 周）

### 5.1 选择 Histogram 还是 Summary

优先使用 Histogram 的典型场景：

- 需要跨实例聚合。
- 需要灵活计算多个分位数。
- 能提前估计延迟分布和 SLO 边界。

Summary 适合：

- 单实例本地观察。
- 分位数目标固定且不能接受服务端计算成本。

在 Go client 中为延迟选择 bucket 时，围绕业务目标设置边界。例如，如果 SLO 是 300ms，则必须在 300ms 附近有足够细的桶，而不是只使用默认桶。

### 5.2 业务指标的边界

推荐：订单创建成功数、支付失败数、队列积压、库存不足数、缓存命中数。

不推荐：每个用户一个序列、完整请求参数、动态错误信息、原始 SQL、任意 trace/span id。需要关联单次请求时使用日志或 Tracing，而不是把高基数数据塞进指标。

### 动手任务

- 为订单服务定义 3 个业务 Counter、1 个队列 Gauge、2 个依赖 Histogram。
- 通过压测生成快、慢、错误三类请求，验证 P95 和 P99 的变化。
- 修改 bucket，比较分位数误差和存储序列数量。
- 写一篇“指标命名与 label 规范”，让队友可以据此新增指标。

## 阶段 6：Alerting Rule 与 Alertmanager（第 13～14 周）

### 6.1 告警设计原则

一条好告警必须回答：发生了什么、影响谁、严重程度、多久后升级、下一步怎么排查。

- 告警应面向用户影响或 SLO，而不是所有异常都告警。
- 使用 `for` 防止瞬时抖动。
- 告警标签用于路由，注释用于人读信息。
- 每条告警配套 Runbook URL 和明确的恢复条件。
- 告警数量、重复通知和无效告警率也要被监控。

### 6.2 示例规则

```yaml
groups:
  - name: order-service-slo
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))

      - record: job:http_errors:rate5m
        expr: sum by (job) (rate(http_requests_total{status=~"5.."}[5m]))

      - alert: OrderServiceHighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{job="go-app",status=~"5.."}[5m]))
            /
            sum(rate(http_requests_total{job="go-app"}[5m]))
          ) > 0.05
        for: 10m
        labels:
          severity: page
          team: backend
        annotations:
          summary: "订单服务 5xx 错误率过高"
          description: "过去 10 分钟错误率超过 5%，当前值={{ $value }}"
          runbook_url: "https://example.invalid/runbooks/order-service-high-error-rate"
```

使用 `promtool test rules` 为规则编写输入序列和预期结果，测试边界值、缺失数据、实例聚合和恢复。

### 6.3 Alertmanager 必学能力

- route：按 `severity`、`team`、`service` 分流。
- group_by：合并同一事故的多条告警。
- group_wait、group_interval、repeat_interval：控制通知节奏。
- inhibit_rules：例如服务整体宕机时抑制下游依赖告警。
- silence：临时维护或已知故障期间静默。
- receiver：邮件、Webhook、企业微信、钉钉、Slack 等。

### 动手任务

1. 触发高错误率告警，验证 Pending -> Firing -> Resolved。
2. 让 Prometheus target 断开，设计 `TargetDown` 告警。
3. 配置同一服务的 page、ticket 两种路由。
4. 添加抑制规则，避免一次故障通知几十条下游告警。
5. 为每条告警写 Runbook：检查入口、常见原因、回滚方式、升级联系人。

### 验收标准

- 你能解释告警为什么触发、为什么没有触发、为什么重复通知。
- 告警规则通过 `promtool check rules` 和 `promtool test rules`。
- 收到的通知包含服务、环境、严重级别、数值、时间窗口和 Runbook。

## 阶段 7：Grafana Dashboard 与可视化（第 15 周）

### Dashboard 设计顺序

一张服务 Dashboard 建议从上到下：

1. SLO/可用性/错误预算。
2. 请求量、错误率、P95/P99。
3. 按 route、status、instance 的拆分。
4. 下游依赖和队列/连接池。
5. Go runtime、CPU、内存、GC、goroutine。
6. 最近告警和部署/变更标记。

### 实践要求

- 使用变量选择环境、集群、namespace、service、instance。
- Panel 的单位、图例、阈值和时间窗口要能帮助排障。
- 高成本查询先做 Recording Rule。
- Dashboard JSON 纳入 Git；通过 provisioning 或代码部署，避免只存在于个人 Grafana。
- 同一指标在 Dashboard、告警和 Runbook 中使用一致的命名和聚合维度。

### 验收标准

给同学一个“订单接口变慢”的事故场景，他能只看 Dashboard 在 5 分钟内回答：影响范围、开始时间、异常 route、是否为下游或资源瓶颈、当前是否恢复。

## 阶段 8：Exporter、服务发现与 Kubernetes（第 16～18 周）

### 8.1 Exporter

先使用成熟 Exporter：node_exporter、blackbox_exporter、mysqld_exporter、redis_exporter、kube-state-metrics。学习其指标语义、权限、采集成本和版本兼容性。

再写一个小型 Go Exporter：

- 从一个 JSON/HTTP 或本地命令读取数据。
- 转换为固定名称、有限 label 的指标。
- 处理超时、部分失败和最后一次成功时间。
- 暴露 Exporter 自身的抓取成功率、耗时和错误计数。
- 为解析逻辑和 Collector 写测试。

### 8.2 服务发现与 relabeling

理解 `static_configs`、`file_sd_configs`、DNS、Kubernetes SD。重点掌握 `relabel_configs` 与 `metric_relabel_configs` 的区别：前者作用于 target，后者作用于抓取到的样本；后者使用不当会浪费抓取和解析成本。

### 8.3 Kubernetes

学习顺序：

1. Pod、Deployment、Service、ConfigMap、Secret、Namespace、RBAC。
2. Prometheus 在集群内如何访问 API Server 和目标 Pod。
3. ServiceMonitor/PodMonitor 的选择器、端口名称、namespaceSelector。
4. Kubernetes 常见指标：容器 CPU/内存、重启、Pod 状态、Deployment 副本数、节点资源。
5. 抓取权限、网络策略、TLS、认证和多租户边界。

### 动手任务

- 用 kind 或 minikube 创建本地集群。
- 部署 Go 服务，暴露 `/metrics`，用 ServiceMonitor 或原生 Kubernetes SD 自动发现。
- 滚动更新服务，观察 Pod 重启、抓取目标变化和告警行为。
- 注入 CPU、内存、延迟和下游失败，验证 Dashboard 和告警。
- 写出“服务未被抓取”的排障清单：标签、端口、路径、权限、网络、target 页面错误。

### 验收标准

- 新增一个带正确标签的 Deployment 后无需修改 Prometheus 主配置即可被发现（在 Operator 方案下）。
- 能从 target 页面和 Kubernetes 资源状态定位发现失败。
- 能解释 Pod 重启后时间序列变化、实例标签变化和告警恢复行为。

## 阶段 9：TSDB、远程存储、HA 与性能（第 19～20 周）

### 必学主题

- 本地 TSDB：block、WAL、压缩、索引、时间序列和样本的关系。
- 数据保留：按时间和按磁盘大小配置，估算容量和增长率。
- 磁盘、CPU、内存、网络对抓取和查询的影响。
- Remote Write/Read 的目标、队列、重试、背压和失败监控。
- 联邦（federation）与集中式远程存储的适用场景。
- HA：双 Prometheus 抓取、去重、告警重复和外部标签设计。
- Thanos、Cortex、Mimir、VictoriaMetrics 等方案的定位和迁移成本。
- 多租户、数据隔离、认证、TLS、网络访问控制和敏感指标治理。

### 容量估算思路

粗略估算要收集：目标数量、每个 target 的 series 数、抓取间隔、样本写入速率、保留时间、压缩后磁盘占用、查询并发和峰值。估算只是起点，必须用实际 TSDB 指标和压测校准。

### 性能实验

- 逐步增加 target 和 label 组合，观察 `prometheus_tsdb_head_series`、抓取样本、内存和 WAL。
- 对复杂 Dashboard 查询做基准，比较原始查询和 Recording Rule。
- 人为制造远程写入失败，观察队列积压、重试和恢复后的行为。
- 记录查询耗时、失败率、规则评估延迟和数据新鲜度。

### 验收标准

- 能给出一份保留 15 天、1000 个 target 的初步容量方案，并列出需要压测验证的假设。
- 能说明单纯增加 Prometheus 副本为什么不等于自动 HA 去重。
- 能在查询慢时判断问题来自查询表达式、数据量、Dashboard 刷新频率还是存储后端。

## 阶段 10：综合毕业项目（第 21～24 周）

### 项目题目

实现一个“订单处理平台可观测性系统”：

```text
API Gateway -> Order API -> Inventory/Payment 模拟依赖
                    |
                    +-> Queue Worker -> PostgreSQL/Redis（可选）
```

### 最低功能要求

- 2～3 个 Go 服务，至少包含 HTTP API 和异步 Worker。
- 每个服务暴露 `/metrics`，具备 RED、依赖、队列和关键业务指标。
- Prometheus 抓取配置、Recording Rules、Alerting Rules 全部进 Git。
- Grafana 至少包含总览、服务详情、依赖和主机/Kubernetes 四张 Dashboard。
- Alertmanager 能按团队和严重级别路由，至少配置一种 Webhook 通知。
- 提供 Docker Compose 启动方式；有条件再提供 Kubernetes manifests 或 Helm chart。
- 提供故障注入脚本：高延迟、5xx、依赖超时、Pod 重启、磁盘空间不足。
- 每条告警有 Runbook；完成至少三次事故复盘。

### 建议的项目指标

- API：请求速率、5xx 比例、P95/P99、in-flight。
- Worker：消费速率、处理耗时、失败数、重试数、队列深度。
- 依赖：连接错误、超时、调用延迟、熔断状态。
- 资源：CPU、内存、GC、goroutine、文件描述符、容器重启。
- 业务：订单创建成功、支付失败、库存不足、最终完成订单数。

### 项目验收清单

- [ ] 全新机器可按 README 在 15 分钟内启动。
- [ ] `go test ./...`、`go test -race ./...` 通过。
- [ ] `promtool check config`、`promtool check rules` 通过。
- [ ] 规则测试覆盖正常、异常、恢复和缺失数据。
- [ ] Dashboard 查询没有无界 label 和明显高成本查询。
- [ ] 任意一个实例宕机时，能识别影响范围并收到正确告警。
- [ ] 产生一次错误率事故并完成从告警到定位的演示。
- [ ] 产生一次延迟事故并判断是应用、依赖还是资源问题。
- [ ] 记录序列数、样本速率、磁盘、内存、查询耗时等容量数据。
- [ ] README 写清启动、配置、故障注入、排障和清理步骤。

## 四、每周学习模板

建议每周安排 5 个学习单元：

| 单元 | 时间 | 内容 |
| --- | ---: | --- |
| 理论 | 1.5 小时 | 阅读官方文档并画出概念关系 |
| 编码 | 2.5 小时 | 修改 Go 服务或写 Exporter |
| 查询 | 2 小时 | 完成 5～10 条 PromQL |
| 运维 | 1.5 小时 | 启停、配置、故障注入、查看日志和 target |
| 复盘 | 0.5～1 小时 | 更新 README、截图、问题和下一步 |

每周交付物固定写成：

```text
本周目标：
完成的实验：
新增的代码/配置：
新增的 PromQL：
观察到的现象：
遇到的问题与证据：
下一周要验证的假设：
```

## 五、必须掌握的排障路径

### 5.1 “Prometheus 抓不到服务”

1. 看 `/targets` 的 `State`、Last Scrape、Last Error。
2. 从 Prometheus 所在网络执行 `curl target:port/metrics`。
3. 检查 DNS、端口、路径、HTTP 状态、TLS 和认证。
4. 检查服务是否监听正确网卡，而不是只监听 `127.0.0.1`。
5. 检查 Kubernetes Service/Pod 标签、端口名、RBAC 和 NetworkPolicy。
6. 检查响应是否为合法 Prometheus exposition format。

### 5.2 “指标数量暴涨”

1. 查看 `prometheus_tsdb_head_series` 和目标的 `scrape_samples_post_metric_relabeling`。
2. 找出最近新增的指标和 label。
3. 检查 URL、用户 ID、订单 ID、错误文本等无界值。
4. 临时限制或移除问题指标，同时修复应用埋点。
5. 评估内存、磁盘、查询和远程写入影响。

### 5.3 “告警没有触发”

1. 在 Graph 页面单独执行告警表达式。
2. 确认数据存在、label 匹配正确、单位正确。
3. 检查 `for` 是否仍处于 Pending。
4. 看 `/rules` 的评估错误和最近评估时间。
5. 检查 Alertmanager route、silence、inhibit 和 receiver。

### 5.4 “Dashboard 很慢”

1. 找到慢 Panel 和实际 PromQL。
2. 缩小时间范围、降低刷新频率、减少无关 label。
3. 先聚合再计算分位数，避免不必要的高基数维度。
4. 将重复查询做成 Recording Rule。
5. 检查 Prometheus 查询并发、内存、磁盘和远程读取。

## 六、常见误区

- 只会装 Grafana，不理解 PromQL 和指标语义。
- 把所有日志字段都做成 label。
- 用 `rate` 处理 Gauge，或用当前 Counter 值直接当速率。
- 只看平均延迟，不看 Histogram 分位数和 SLO。
- 告警只写阈值，不写 `for`、影响范围和 Runbook。
- 把 Prometheus 当日志库、Tracing 系统或高基数明细数据库。
- 只在本地静态配置，没练习服务发现、权限和网络。
- Dashboard 和规则直接在 UI 修改，无法复现、审查和回滚。
- 没有观察 Prometheus 自身健康状态，却把它当作绝对可靠的监控源。

## 七、学习资源顺序

优先使用官方资料，阅读时以“读一节、做一个实验”为单位：

1. Prometheus 官方文档：概念、配置、查询、告警、存储和 HTTP API。
2. PromQL 函数与操作符参考：边查边写查询，不背完整语法。
3. `client_golang` 文档和示例：Registry、Collector、promhttp、Histogram。
4. Grafana 官方文档：数据源、变量、Dashboard as Code、告警。
5. Alertmanager 官方文档：路由、分组、抑制、静默和通知。
6. Kubernetes/Prometheus Operator 文档：ServiceMonitor、PodMonitor、RBAC 和服务发现。
7. Prometheus 社区 Exporter 文档：先理解成熟实现，再动手写自己的 Exporter。

阅读第三方文章或视频时，用官方文档验证版本、配置字段和安全建议；不要复制无法解释的 YAML。

## 八、阶段性自测题

每完成一个阶段，闭卷回答：

1. Prometheus 为什么默认使用 Pull 模型？哪些情况适合 Pushgateway？
2. Counter、Gauge、Histogram、Summary 分别解决什么问题？
3. 为什么 `rate(counter[5m])` 比直接读取 Counter 更有意义？
4. Histogram 跨实例聚合时为什么要保留 `le`？
5. 什么是时间序列基数？哪个 label 最容易造成事故？
6. `relabel_configs` 与 `metric_relabel_configs` 有什么区别？
7. target 健康但业务错误率很高，`up` 会是什么值？
8. target 宕机时，为什么需要 `absent` 或 `up == 0` 类告警？
9. `for`、group_wait、repeat_interval 分别控制什么？
10. Prometheus 双副本如何避免告警和查询重复？
11. 什么情况下应该使用 Recording Rule？
12. 如何判断一个慢查询是表达式问题还是存储/资源问题？

答不出来的题目要回到实验，而不是只记答案。

## 九、作品集与求职表达

把最终项目整理成可公开展示的仓库，至少包括：

- 一张架构图和数据流图。
- 一键启动命令和版本信息。
- 指标规范、label 选择理由和基数估算。
- 5～10 条关键 PromQL 的解释。
- Dashboard 截图或 JSON provisioning。
- 告警规则、通知样例和 Runbook。
- 一次错误率事故、一次延迟事故的时间线和根因分析。
- 容量估算、压测数据、已知限制和后续改进。

面试时不要只说“我会 Prometheus”。应能完整讲清一个场景：

```text
用户反馈接口变慢
  -> Dashboard 确认影响范围和开始时间
  -> PromQL 拆分 route/status/instance
  -> Histogram 判断延迟分布
  -> 依赖指标确认下游是否异常
  -> 资源指标排除 CPU/内存/GC/连接池
  -> 日志/Tracing 进行单请求定位
  -> 告警、缓解、回滚和复盘
```

## 十、最终完成标准

你可以把“学会 Prometheus”定义为以下可观察结果，而不是课程数量：

- 能独立为一个 Go 服务设计指标并解释每个 label。
- 能在 30 分钟内搭建本地监控栈并让服务被抓取。
- 能在 10 分钟内写出请求量、错误率和 P95 查询。
- 能把一条无效告警改造成带 SLO、`for`、路由和 Runbook 的告警。
- 能从 target、规则、查询、应用和 Kubernetes 五个层面排查问题。
- 能说明监控系统自身的容量、可靠性、安全边界和升级方案。
- 能用一个可复现项目证明以上能力。

从现在开始，第一周只做三件事：完成阶段 0 的 Go 服务、建立实验仓库结构、写出阶段 1 的指标设计表。下一周再接入 Prometheus；这样每个后续查询和告警都有真实业务上下文。
