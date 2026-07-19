# 故障注入、告警演练与 Runbook

## 1. 故障注入原则

故障注入只允许在本地或明确隔离的环境：

- 默认关闭。
- 需要显式环境变量或配置开关。
- 记录注入开始、结束和操作者。
- 不把注入参数放进高基数 label。
- 注入后必须有恢复命令和验证查询。

建议 fake dependency 支持：

```text
INJECT_DELAY_MS=0
INJECT_ERROR_RATE=0
INJECT_STATUS_CODE=200
```

Worker 支持：

```text
INJECT_PROCESSING_DELAY_MS=0
INJECT_CONSUMER_PAUSE=false
```

## 2. 场景 A：依赖延迟

步骤：

1. 将支付依赖延迟设置为 800ms。
2. 发送一批订单请求。
3. 查看依赖 Histogram P95 和 API P95。
4. 确认 API 错误率不一定增加，但用户体验变差。
5. 恢复延迟并观察规则恢复。

查询：

```promql
histogram_quantile(
  0.95,
  sum by (le, service, operation) (
    rate(order_service_dependency_duration_seconds_bucket[5m])
  )
)
```

结论必须说明：应用 P95 和依赖 P95 的差异，以及是否存在超时边界。

## 3. 场景 B：支付 5xx

步骤：

1. 设置支付依赖返回 500。
2. 发送成功和失败订单。
3. 查看 `dependency_requests_total`、`http_requests_total` 和业务 Counter。
4. 验证 5xx 告警进入 Pending/Firing。
5. 检查 Alertmanager 收到的标签和 Runbook。
6. 恢复依赖，验证 Resolved。

## 4. 场景 C：Worker 停止消费

步骤：

1. 暂停 Worker。
2. 继续创建订单。
3. 观察 `queue_depth`、生产/消费速率和处理延迟。
4. 验证队列积压告警。
5. 恢复 Worker，观察积压下降和告警恢复。

不要只观察绝对深度，还要比较：

```promql
rate(order_worker_messages_total{result="success"}[5m])
```

与生产速率之间的差异。

## 5. 场景 D：target down

```bash
docker compose stop order-api
```

验证：

```promql
up{job="order-api"}
```

检查：

- TargetDown 是否触发。
- API 内部延迟告警是否被合理抑制。
- Alertmanager 是否按 service 分组。
- 恢复后是否收到 Resolved。

## 6. 场景 E：高基数回归

在实验分支中加入：

```go
requests.WithLabelValues(r.URL.Query().Get("user_id")).Inc()
```

生成大量用户 ID，观察：

```promql
prometheus_tsdb_head_series
scrape_samples_scraped{job="order-api"}
```

修复为有限 label 后重新测量，并在复盘中写出：

- 发生原因。
- 为什么 code review 没拦截。
- 序列峰值。
- 内存和查询影响。
- 如何增加自动化保护。

## 7. Runbook 实例：OrderAPIHighErrorRate

```text
影响：
订单创建可能失败，优先确认用户影响和失败 route。

第一步：
打开 Order Service Dashboard，查看 5xx、route、instance 和依赖错误。

第二步：
执行：
sum by (service, operation, result) (
  rate(order_service_dependency_requests_total{result!="success"}[5m])
)

第三步：
检查 fake-dependency 或真实依赖日志，确认是网络、超时还是返回码。

临时缓解：
降低非核心重试、启用降级、回滚最近发布或切换依赖。

恢复验证：
错误率连续 15 分钟低于阈值，API 和依赖 target 均为 UP，业务成功率恢复。

升级：
超过 15 分钟未恢复，通知 backend on-call；涉及支付时通知 payment team。
```

## 8. 事故复盘模板

```text
标题：
影响时间：
用户影响：
检测方式：
时间线：
根因：
促成因素：
做得好的地方：
做得不好的地方：
缺失的指标/日志/Tracing：
告警问题：
修复项：
负责人和截止时间：
如何验证改进：
```

## 9. 演练验收

至少完成三次演练：

- 依赖延迟。
- 依赖错误。
- Worker 停止或 target down。

每次都提交告警状态、关键查询、截图、根因和恢复证据。

