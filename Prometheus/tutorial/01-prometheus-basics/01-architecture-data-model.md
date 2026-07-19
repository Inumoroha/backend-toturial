# 架构、数据模型与抓取模型

## 1. Prometheus 的核心组件

```text
目标服务 / Exporter
        |  HTTP exposition format
        v
Prometheus server
  ├─ service discovery / scrape
  ├─ TSDB + WAL
  ├─ PromQL 查询
  ├─ recording / alerting rules
  └─ Alertmanager client
        |
        +--> Grafana / HTTP API
        +--> Alertmanager --> 通知渠道
```

Prometheus server 内部最重要的工作是：

1. 根据配置发现 target。
2. 定时向 target 发 HTTP 请求。
3. 解析样本并写入 TSDB。
4. 按规则评估表达式。
5. 为查询和 Grafana 提供时间序列。

## 2. Pull 的含义

Pull 模型意味着 Prometheus 主动决定：

- 抓取哪些 target。
- 多久抓取一次。
- 超时时间是多少。
- 抓取失败如何标记。
- 目标是否暂时不可达。

被监控服务只负责暴露当前指标。这样 Prometheus 可以通过 target 健康状态发现服务不可达，不需要每个服务维护一个向监控系统推送的客户端连接。

Pushgateway 只适合短生命周期批处理任务等特殊场景。不要把长期运行服务都通过 Pushgateway 推送，否则会丢失 target 生命周期语义和下线信息。

## 3. 数据模型

时间序列由指标名和 label 集合唯一确定：

```text
http_requests_total{job="go-app",instance="app:8080",method="GET",route="/orders",status="200"}
```

其中：

- `job` 通常来自 scrape 配置。
- `instance` 通常表示 target 地址。
- 业务 label 由应用定义。
- 每个不同 label 组合都是一条独立时间序列。

同一个指标名加上不同 `route` 或 `status`，不会覆盖，而是产生多条序列。序列数量约等于各 label 值数量的乘积，因此 label 设计是容量设计的一部分。

## 4. 指标类型

### Counter

只增不减的累计量，例如请求总数、错误总数、重试总数。进程重启后可以归零，查询时使用 `rate` 或 `increase`。

### Gauge

可以上升或下降的当前状态，例如队列深度、温度、内存、并发请求数。不要对 Gauge 使用 `rate` 来表示业务速率。

### Histogram

把观察值分桶，并产生 `_bucket`、`_sum`、`_count`。服务端可以用 `histogram_quantile` 计算分位数，多个实例可以先聚合桶再计算。

### Summary

客户端计算分位数和总量。单实例观察方便，但分位数通常不能像 Histogram 那样直接跨实例聚合。

## 5. 样本、时间范围和缺失

一个样本是时间戳和值。查询结果可能是：

- instant vector：某个时刻的一组序列。
- range vector：每条序列在时间窗口中的样本。
- scalar：单个数值。
- 没有结果：可能是没有数据、label 不匹配、target 已停止或查询窗口不对。

“没有结果”和“值为 0”不是同一件事。告警和 Dashboard 必须明确需要哪一种语义。

## 6. Prometheus 自监控

启动后优先查询：

```promql
up
prometheus_tsdb_head_series
scrape_samples_scraped
scrape_duration_seconds
prometheus_rule_evaluation_failures_total
```

Prometheus 自身也会被抓取。监控监控系统，才能知道你看到的“没有数据”是否其实是 Prometheus 自己已经不健康。

## 7. 动手实验

1. 创建两个相同应用 target，只修改 instance。
2. 给应用增加一个 `status` label，观察序列数量变化。
3. 删除一个 target，观察旧数据是否立即消失。
4. 停止 target，查询 `up` 和 `rate`。
5. 重启 Prometheus，观察本地数据和 rule 状态。

## 验收问题

- 为什么同一个指标名可以出现很多条时间序列？
- 为什么 `up == 1` 不代表业务请求一定成功？
- 为什么 Histogram 能聚合而 Summary 的分位数通常不能简单聚合？
- 为什么一个 target 的 label 变化可能导致新旧时间序列同时存在？

