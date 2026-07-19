# Counter、速率、聚合与错误率

## 1. 为什么不能直接看 Counter

`http_requests_total` 代表进程启动以来累计请求数。实例启动时间不同，直接相加可以得到累计总量，但不能直接回答“现在每秒多少请求”。

```promql
rate(http_requests_total[5m])
```

`rate` 会计算窗口内的每秒平均增长，并处理常见 Counter reset。

`increase` 更适合回答窗口累计：

```promql
increase(http_requests_total[1h])
```

它近似于过去一小时增加了多少次请求。

## 2. 先按实例，再按服务聚合

最常见的错误率：

```promql
sum(rate(http_requests_total{job="go-app",status=~"5.."}[5m]))
/
sum(rate(http_requests_total{job="go-app"}[5m]))
```

先对所有实例求和，再相除，得到服务级错误率。不要先计算每个实例的比率再简单平均，因为各实例流量可能不同。

按 route 保留维度：

```promql
sum by (route) (
  rate(http_requests_total{job="go-app",status=~"5.."}[5m])
)
/
sum by (route) (
  rate(http_requests_total{job="go-app"}[5m])
)
```

分子和分母必须保留完全相同的 label 集合，否则向量运算无法匹配或会得到错误结果。

## 3. 低流量和除零

低流量 route 可能分母为 0 或没有样本。处理方式取决于用途：

- Dashboard 可以显示空值，提醒没有流量。
- 告警要使用 `and`、最小请求量条件或 SLO 规则避免噪音。
- 不要用 `or vector(0)` 把所有情况都隐藏。

示例：

```promql
(
  sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
  sum(rate(http_requests_total[5m]))
) > 0.05
and
sum(rate(http_requests_total[5m])) > 1
```

阈值 1 的单位是每秒请求，真实项目需要根据业务流量调整。

## 4. Counter reset 实验

1. 让 Go 服务收到请求。
2. 查询 `http_requests_total` 和 `rate(...)`。
3. 重启服务。
4. 观察 Counter 回到较小值。
5. 观察 `rate` 是否出现异常尖峰。

如果应用在不重启时手动把 Counter 减小，Prometheus 可能把它当作 reset；这会让结果失真。应用必须保证 Counter 单调递增。

## 5. 聚合修饰符

```promql
sum by (route) (...)
sum without (instance, pod) (...)
count by (status) (...)
topk(5, sum by (route) (...))
```

`by` 表示保留哪些 label；`without` 表示移除哪些 label。写完后用：

```promql
count(sum by (route) (rate(http_requests_total[5m])))
```

检查序列数量是否符合预期。

## 6. 常用运维查询

```promql
# 每个服务的请求速率
sum by (job) (rate(http_requests_total[5m]))

# 每个 route 的错误率
sum by (route) (rate(http_requests_total{status=~"5.."}[5m]))
/
sum by (route) (rate(http_requests_total[5m]))

# 状态码分布
sum by (status) (rate(http_requests_total[5m]))

# 过去一天每个实例的进程启动时间变化次数（近似重启次数）
changes(process_start_time_seconds[24h])
```

`process_start_time_seconds` 是进程启动时间的 Gauge，不应该用 `increase` 计算重启次数；重启时它会跳到一个新的时间戳，`changes` 可以统计窗口内发生了多少次变化。生产环境最好同时结合容器编排平台的重启计数，并注意 Prometheus 抓取中断会让结果不完整。

## 7. 练习

构造 3 个实例：

- A：100 req/s，5% 错误。
- B：10 req/s，0% 错误。
- C：1 req/s，100% 错误。

比较：

1. 全局加权错误率。
2. 三个实例错误率的算术平均。
3. 按实例展示的错误率。

你应能解释为什么服务级错误率不能简单平均实例比率。

## 8. 验收

能够从一个原始 Counter 开始，逐步写出：

```text
原始序列
-> 每秒速率
-> 按 route 聚合
-> 错误分子/总量分母
-> 低流量保护
-> 告警条件
```
