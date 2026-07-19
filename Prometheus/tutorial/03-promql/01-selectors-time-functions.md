# 选择器、时间范围与基础运算

## 1. 四种基本结果

### Instant vector

```promql
up
go_goroutines
http_requests_total{job="go-app"}
```

返回当前评估时刻每条序列的最新值。

### Range vector

```promql
http_requests_total[5m]
```

返回每条序列过去 5 分钟的样本，不能直接在图表上当作一个数展示，通常交给 `rate`、`increase` 等函数。

### Scalar

```promql
5
vector(0)
```

可用于比较、填充和计算。

### 空结果

```promql
up{job="does-not-exist"}
```

空结果可能代表确实没有序列、selector 写错、时间窗口之外没有样本或 target 已不再被抓取。不要自动把空结果当 0。

## 2. Label matcher

```promql
up{job="go-app"}
up{job!="go-app"}
up{status=~"2.."}
up{route!~"/healthz|/metrics"}
```

正则 matcher 必须匹配完整 label 值的语义。过滤前先确认 label 真的存在：

```promql
count by (job, route) (http_requests_total)
```

## 3. 算术和比较

```promql
rate(http_requests_total[5m]) * 60
go_goroutines > 100
http_request_duration_seconds_sum / http_request_duration_seconds_count
```

表达式计算前检查单位。Counter 的 `rate` 是每秒，乘以 60 才是每分钟；时间的 `_sum / _count` 是平均秒数，不是 P95。

## 4. 时间窗口

窗口太短：

- 对低流量服务，`rate(...[1m])` 可能样本太少、图线抖动。
- scrape interval 为 15s 时，5m 通常至少包含多个样本。

窗口太长：

- 会掩盖刚发生的事故。
- 告警恢复变慢。

常见组合：

```promql
rate(http_requests_total[5m])
increase(http_requests_total[1h])
avg_over_time(order_queue_depth[15m])
max_over_time(order_queue_depth[1h])
```

## 5. `offset` 和对比

```promql
sum(rate(http_requests_total[5m]))
  - sum(rate(http_requests_total[5m] offset 1h))
```

比较前确认两个窗口的 label 集合、采样稳定性和业务周期。不要把节假日、发布时段和正常时段直接比较后下结论。

## 6. 空值处理

展示场景可以：

```promql
sum(rate(http_requests_total{status=~"5.."}[5m])) or vector(0)
```

告警场景要谨慎。`or vector(0)` 可能把抓取失败伪装成“错误率为 0”。更好的做法是同时监控 `up` 和数据新鲜度。

## 7. 常用函数速查

```promql
sort_desc(sum by (route) (rate(http_requests_total[5m])))
topk(5, sum by (route) (rate(http_requests_total[5m])))
clamp_min(order_queue_depth, 0)
round(go_goroutines, 1)
count by (job) (up)
changes(process_start_time_seconds[24h])
```

使用函数前先确认输入类型和单位。例如 `sort_desc` 只改变展示顺序，不能改变时间序列含义；`clamp_min` 适合清理展示中的负值，但不能修复产生负值的应用；`changes` 统计的是值变化次数，不一定等于真实重启次数。

## 8. 实验：零值、空值和抓取中断

1. 创建一个存在但值为 0 的 Gauge，查询它并记录结果。
2. 删除该指标，查询同一个 selector，记录空结果。
3. 停止 target，观察 `up`、原始业务指标和 `absent`。
4. 用 `or vector(0)` 处理 Dashboard，再对比告警表达式不使用它的结果。
5. 写出一个不会把抓取失败伪装成健康的表达式。

## 9. 练习

写出以下查询并记录单位：

1. 当前 app target 是否健康。
2. 每个 route 的当前请求速率。
3. 最近一小时订单创建数。
4. 当前最大队列深度。
5. 过去 15 分钟平均队列深度。
6. 当前没有被抓取的 app target。

## 10. 验收

你要能解释每个表达式：

- 输入是什么类型。
- 输出是什么类型。
- 时间窗口如何影响结果。
- label 是否被保留。
- 没有数据时用户会看到什么。
