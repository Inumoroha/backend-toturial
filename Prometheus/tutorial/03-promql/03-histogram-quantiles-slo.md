# Histogram、分位数与 SLO

## 1. Histogram 暴露什么

指标名：

```text
http_request_duration_seconds
```

会产生：

```text
http_request_duration_seconds_bucket{le="0.1"}
http_request_duration_seconds_bucket{le="0.2"}
http_request_duration_seconds_bucket{le="+Inf"}
http_request_duration_seconds_sum
http_request_duration_seconds_count
```

`_bucket` 是累计桶：`le="0.2"` 包含小于等于 0.2 秒的观察值，`+Inf` 包含全部观察值。

## 2. 平均值不是分位数

平均值：

```promql
rate(http_request_duration_seconds_sum[5m])
/
rate(http_request_duration_seconds_count[5m])
```

平均值可能掩盖长尾。P95：

```promql
histogram_quantile(
  0.95,
  sum by (le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

按 route：

```promql
histogram_quantile(
  0.95,
  sum by (le, route) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

按服务聚合时保留 `le`；没有 `le` 就无法让 `histogram_quantile` 识别桶边界。

## 3. bucket 和估计误差

Histogram 分位数是根据桶边界估计的。如果 P95 落在 0.2 到 0.5 秒之间，结果无法知道桶内部的真实分布。

设计原则：

- SLO 边界附近使用足够细的桶。
- 关注短请求时不要只用秒级大桶。
- 关注长尾时保留更高的桶。
- bucket、label 和实例数量共同决定序列成本。

## 4. SLO 达标率

如果 SLO 是 300ms，可以使用满足桶的请求数近似：

```promql
sum(rate(http_request_duration_seconds_bucket{le="0.3"}[5m]))
/
sum(rate(http_request_duration_seconds_count[5m]))
```

这表示过去窗口内不超过 300ms 的请求比例。确认你的 bucket 中确实存在 `le="0.3"`，且分子分母保留相同的 service、route、job 维度。

## 5. Burn rate 思维

错误预算告警通常使用短窗口和长窗口组合：

```promql
(
  error_ratio_5m > 14.4 * error_budget_ratio
)
and
(
  error_ratio_1h > 14.4 * error_budget_ratio
)
```

实际项目还要定义：

- SLO 窗口。
- budget ratio。
- page 和 ticket 的不同阈值。
- 低流量时的处理。
- 告警 `for` 时间。

不要只抄数字。先写出你的 SLO 和预算计算。

## 6. Summary 的限制

Summary 在客户端输出 `quantile` label，例如：

```text
http_request_duration_seconds{quantile="0.95"}
```

不同实例的 P95 不能简单相加、平均或再次计算全局 P95。需要跨实例聚合时优先 Histogram。

## 7. 实验

1. 使用固定延迟生成 1000 个请求。
2. 改变 bucket，观察 P95 的估计。
3. 让一个实例变慢，观察全局 P95。
4. 比较平均值和 P99，解释长尾。
5. 用 SLO bucket 查询计算达标率。

## 8. 验收

提交：

- bucket 选择依据。
- P50、P95、P99 查询。
- SLO 达标率查询。
- 平均值误导性的一个具体例子。
- Histogram 序列成本估算。

