# Middleware、依赖指标和指标测试

## 1. 一个可测试的 middleware

```go
func Instrument(m *Metrics, route string, next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		m.InFlight.WithLabelValues(route).Inc()
		defer m.InFlight.WithLabelValues(route).Dec()

		sw := &statusWriter{ResponseWriter: w}
		next.ServeHTTP(sw, r)

		status := sw.status
		if status == 0 {
			status = http.StatusOK
		}
		m.Requests.WithLabelValues(r.Method, route, strconv.Itoa(status)).Inc()
		m.Duration.WithLabelValues(r.Method, route).
			Observe(time.Since(start).Seconds())
	})
}
```

这段代码适合学习，不代表所有 ResponseWriter 接口都被完整包装。对于真实项目，必须测试 streaming、hijacking、redirect 和 panic 场景，或采用成熟 middleware。

## 2. 依赖指标

依赖指标要记录“实际调用”，而不是业务意图：

```go
type DependencyMetrics struct {
	Requests *prometheus.CounterVec
	Duration *prometheus.HistogramVec
}

func (m *DependencyMetrics) Observe(
	service, operation, result string,
	start time.Time,
) {
	m.Requests.WithLabelValues(service, operation, result).Inc()
	m.Duration.WithLabelValues(service, operation).
		Observe(time.Since(start).Seconds())
}
```

`result` 使用有限枚举，例如：

```text
success
timeout
connection_error
http_4xx
http_5xx
canceled
```

不要把远端返回的错误文本作为 result。

## 3. 业务指标

业务指标必须与领域动作绑定：

```go
ordersTotal := prometheus.NewCounterVec(
	prometheus.CounterOpts{
		Namespace: "order_service",
		Name:      "orders_total",
		Help:      "Total orders by final processing result.",
	},
	[]string{"result"},
)
```

建议 result 的值域在代码中固定，并在文档中公开：

```text
created
rejected
payment_failed
inventory_failed
completed
```

若订单状态可以不断增加，不要把每个外部系统自由传入的状态直接当 label；应该先映射到稳定的有限集合。

## 4. 指标测试

### 检查文本输出

```go
func TestMetricsExposeExpectedNames(t *testing.T) {
	m := NewMetrics()
	m.Requests.WithLabelValues("GET", "/orders", "200").Add(3)

	gathered, err := testutil.GatherAndCount(
		m.Registry,
		"order_service_http_requests_total",
	)
	if err != nil {
		t.Fatal(err)
	}
	if gathered != 1 {
		t.Fatalf("metric families = %d, want 1", gathered)
	}
}
```

### 检查样本值

```go
func TestRequestCounterValue(t *testing.T) {
	m := NewMetrics()
	m.Requests.WithLabelValues("GET", "/orders", "201").Add(2)

	if got := testutil.ToFloat64(
		m.Requests.WithLabelValues("GET", "/orders", "201"),
	); got != 2 {
		t.Fatalf("counter = %v, want 2", got)
	}
}
```

### 检查 exposition format

将 handler 绑定到 `httptest.NewRecorder`，再用 `expfmt.TextParser` 或 `testutil.CollectAndCompare` 检查名称和 label。测试必须覆盖：

- 没有 label 值时是否错误。
- label 数量顺序是否稳定。
- Histogram 的 `_bucket`、`_sum`、`_count` 是否出现。
- 没有请求时是否暴露预期的零值或不暴露。
- 指标是否注册了两次。

## 5. 高基数防护

在代码 review 中建立硬规则：

- 原始路径必须经过 route normalization。
- label 值来自枚举或配置，而不是请求输入。
- 新增 label 必须给出最大值域和序列估算。
- Histogram 新增 bucket 要说明存储成本。
- 业务异常细节写日志，指标只保留可聚合分类。

可以把序列预算写入设计文档：

```text
单实例 HTTP 指标：
methods(5) × routes(30) × statuses(8) = 1,200
10 个实例 = 12,000 条
Histogram 11 个 bucket 会产生额外 bucket 序列和 sum/count
```

## 6. `/metrics` 安全

选择一种方案并写入部署文档：

- 只在集群内部监听和访问。
- 使用独立 metrics 端口 + NetworkPolicy。
- 在反向代理层做 mTLS 或认证。
- 不在指标中暴露 token、邮箱、完整 SQL 和敏感配置。
- 对 exporter 的命令执行、文件读取和 HTTP 出站做最小权限限制。

## 7. 阶段验收

提交：

1. 指标设计表。
2. middleware 测试。
3. 依赖和业务指标测试。
4. 高基数实验结果。
5. `/metrics` 访问边界说明。

当测试可以在 CI 中稳定运行后，再进入 PromQL；否则查询出来的数字没有可靠基础。

