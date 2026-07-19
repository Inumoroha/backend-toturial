# 编写一个可靠的 Go Exporter

## 1. Exporter 的职责边界

Exporter 负责把外部系统的状态转换为 Prometheus 指标。它不应该：

- 把每一条原始记录无条件转成时间序列。
- 无限等待外部 API。
- 把外部错误文本直接变成 label。
- 在一次抓取失败时返回上一轮数据却不告诉 Prometheus。
- 使用超出部署权限的文件或命令。

推荐暴露三类指标：

```text
inventory_exporter_scrape_success
inventory_exporter_scrape_duration_seconds
inventory_exporter_last_success_timestamp_seconds
```

以及业务数据：

```text
inventory_stock_units{category="book"}
```

## 2. Collector 结构

```go
type Collector struct {
	client      *http.Client
	target      string
	stock       *prometheus.Desc
	success     *prometheus.Desc
	duration    *prometheus.Desc
	lastSuccess *prometheus.Desc
	errors      *prometheus.Desc
	state       *scrapeState
}

func NewCollector(target string) *Collector {
	return &Collector{
		client: &http.Client{Timeout: 3 * time.Second},
		target: target,
		state:  &scrapeState{},
		stock: prometheus.NewDesc(
			"inventory_stock_units",
			"Current inventory units by bounded category.",
			[]string{"category"}, nil,
		),
		success: prometheus.NewDesc(
			"inventory_exporter_scrape_success",
			"Whether the latest inventory scrape succeeded.",
			nil, nil,
		),
		duration: prometheus.NewDesc(
			"inventory_exporter_scrape_duration_seconds",
			"Time spent fetching inventory.",
			nil, nil,
		),
		lastSuccess: prometheus.NewDesc(
			"inventory_exporter_last_success_timestamp_seconds",
			"Unix timestamp of the last successful inventory scrape.",
			nil, nil,
		),
		errors: prometheus.NewDesc(
			"inventory_exporter_scrape_errors_total",
			"Total inventory scrape errors.",
			nil, nil,
		),
	}
}
```

`Describe` 可以发送 descriptors，`Collect` 在每次 Prometheus 抓取时执行外部请求。外部调用必须有 timeout。生产实现还需要在 Collector 外层维护成功时间和错误计数，或者通过一个带锁的状态对象暴露它们；不要在 `Collect` 中启动后台 goroutine，否则一次 Prometheus scrape 可能读到不一致状态。

实现 `Describe` 时把所有 descriptor 发送出去：

```go
func (c *Collector) Describe(ch chan<- *prometheus.Desc) {
	ch <- c.stock
	ch <- c.success
	ch <- c.duration
	ch <- c.lastSuccess
	ch <- c.errors
}
```

如果 Collector 的 descriptor 集合是动态的，可以使用 unchecked collector，但要明确它会减少 Registry 对描述一致性的检查；固定的 Exporter 优先使用可描述的 Collector。

## 3. 采集流程

```go
func (c *Collector) Collect(ch chan<- prometheus.Metric) {
	start := time.Now()
	success := 1.0

	data, err := c.fetch()
	if err != nil {
		success = 0
	}

	if err == nil {
		for _, item := range data {
			ch <- prometheus.MustNewConstMetric(
				c.stock,
				prometheus.GaugeValue,
				float64(item.Units),
				item.Category,
			)
		}
	}

	ch <- prometheus.MustNewConstMetric(
		c.success,
		prometheus.GaugeValue,
		success,
	)
	ch <- prometheus.MustNewConstMetric(
		c.duration,
		prometheus.GaugeValue,
		time.Since(start).Seconds(),
	)
	// 真实实现应从并发安全的状态对象读取 lastSuccess 和 errors。
}
```

关键语义：如果外部系统失败，业务数据不应无提示地继续暴露旧值。可以选择不暴露本次业务样本，同时将 `scrape_success` 置 0，并在 Prometheus 中告警。`last_success_timestamp_seconds` 用于判断“最后一次成功距离现在多久”，`scrape_errors_total` 用于观察失败趋势；两者应在成功/失败状态更新时保持并发安全。

一个最小的并发安全状态对象可以这样设计：

```go
type scrapeState struct {
	mu          sync.RWMutex
	lastSuccess time.Time
	errors      uint64
}

func (s *scrapeState) recordSuccess(t time.Time) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.lastSuccess = t
}

func (s *scrapeState) recordError() {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.errors++
}

func (s *scrapeState) snapshot() (time.Time, uint64) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	return s.lastSuccess, s.errors
}
```

`Collect` 成功时调用 `recordSuccess`，失败时调用 `recordError`，然后把快照转换成常量指标：

```go
lastSuccess, errors := c.state.snapshot()
if !lastSuccess.IsZero() {
	ch <- prometheus.MustNewConstMetric(
		c.lastSuccess,
		prometheus.GaugeValue,
		float64(lastSuccess.Unix()),
	)
}
ch <- prometheus.MustNewConstMetric(
	c.errors,
	prometheus.CounterValue,
	float64(errors),
)
```

这样可以分别告警“最近一次抓取失败”和“失败次数持续增加”，而不是把业务数据是否存在当成唯一信号。

## 4. 限制 label 基数

假设库存接口返回几十万商品，不要使用：

```text
inventory_stock_units{product_id="..."}
```

除非产品集合有明确上限、容量预算和查询需求。可改为：

```text
inventory_stock_units{category="book"}
inventory_stock_units{warehouse="east"}
```

或者只暴露 top/aggregate 数据，其余明细放到日志、数据库或专用查询系统。

## 5. Exporter 自身端点

```go
mux := http.NewServeMux()
mux.Handle("/metrics", promhttp.Handler())
mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
})
http.ListenAndServe(":9115", mux)
```

若 exporter 支持动态 target，应使用白名单或受控配置，不允许用户通过 URL 让 exporter 访问任意内网地址，避免 SSRF。

## 6. 测试

测试：

- HTTP 200 和合法 JSON。
- HTTP 500。
- JSON 解析失败。
- 超时。
- 空结果。
- 部分字段缺失。
- category 不在允许集合中。
- 指标文本包含成功和耗时指标。
- 失败时业务数据的语义符合文档。

## 7. 阶段验收

启动一个模拟库存 API，停止它，再观察：

- exporter 的抓取响应。
- `inventory_exporter_scrape_success`。
- Prometheus target 状态。
- 业务指标是否被错误地保留。
- 告警是否能在合理时间内触发。
