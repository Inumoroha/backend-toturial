# 指标类型、命名和基数

## 1. 先回答“我要知道什么”

不要从指标类型开始，而要从问题开始：

| 问题 | 结果形态 | 常见类型 |
| --- | --- | --- |
| 过去 5 分钟每秒处理多少请求？ | 累计量的变化率 | Counter |
| 现在队列有多少条消息？ | 当前状态 | Gauge |
| 请求延迟分布如何？ | 分桶样本 | Histogram |
| 单实例某个固定分位数是多少？ | 客户端聚合结果 | Summary |

如果一个值既有累计意义又有当前状态意义，通常应该暴露两个指标，而不是让一个指标承担两种语义。

## 2. 命名规范

推荐：

```text
order_service_http_requests_total
order_service_http_request_duration_seconds
order_service_db_connection_pool_in_use
order_service_payload_size_bytes
```

规则：

- 前缀表达应用或领域。
- 名称表达测量对象。
- `_total` 只用于 Counter。
- 时间统一使用 seconds，大小统一使用 bytes。
- 不把 label 值写入名称，例如不要生成 `orders_failed_payment_total`。
- Help 文本说明“测量什么”，不要写“这是一个计数器”。

## 3. Label 的设计

label 是查询维度，也是成本乘数。假设：

```text
method: 4 个值
route: 20 个值
status: 6 个值
instance: 10 个实例
```

理论最大序列约为：

```text
4 × 20 × 6 × 10 = 4,800
```

这只是一个指标的上限，还要乘以 Histogram 的桶数和其他指标数量。

适合 label：

- `method`：有限集合。
- `route`：路由模板，如 `/orders/:id`，不是原始 URL。
- `status`：标准状态码或有限业务结果。
- `service`、`operation`：由代码控制的枚举。

危险 label：

- `user_id`、`order_id`、`trace_id`。
- 原始 URL、查询参数、搜索词。
- `error.Error()`、SQL 文本、异常堆栈。
- 任意外部请求头。
- 动态租户名（除非有严格的租户上限和容量预算）。

## 4. Histogram 桶怎么选

桶应该围绕 SLO 和真实分布选择。例如 HTTP SLO 是 300ms，可以使用：

```go
Buckets: []float64{
    0.005, 0.01, 0.025, 0.05, 0.1,
    0.2, 0.3, 0.5, 1, 2, 5,
}
```

不要只复制默认 bucket。先压测收集分布，再在关注区间增加边界。桶越多、label 越多，序列数量越高。

## 5. Counter 重启和单调性

Counter 进程重启后可以回到 0。Prometheus 的 `rate` 和 `increase` 会处理常见 reset，但如果应用在运行中手动把 Counter 减小，查询会产生错误解释。

不要把 Counter 用作：

- 当前库存。
- 当前队列深度。
- 当前连接数。

这些应该是 Gauge。

## 6. 设计练习

为以下场景各选择类型，并说明 label：

1. 支付调用结果。
2. 当前待处理订单。
3. 订单接口 P99。
4. Go runtime goroutine。
5. 过去一天完成订单数。

## 7. 基数实验

先写一个实验指标：

```go
bad.WithLabelValues(strconv.Itoa(userID)).Inc()
```

生成 10、100、1000、10000 个用户值，观察序列增长。然后改成：

```go
orders.WithLabelValues("success").Inc()
```

对比：

- 内存。
- `scrape_samples_scraped`。
- `prometheus_tsdb_head_series`。
- 查询速度。
- 运维可解释性。

## 8. 阶段验收

提交一份指标规范：

- 指标名称。
- 类型和单位。
- label 允许值。
- 理论序列上限。
- 采集位置。
- 主要查询和告警。
- 禁止加入的字段。
- 变更审核人。

