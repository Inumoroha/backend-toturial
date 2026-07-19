# Go client 与 HTTP 埋点

## 1. 安装依赖

```bash
go get github.com/prometheus/client_golang
go mod tidy
```

`client_golang` 的几个核心对象：

- `Counter`、`Gauge`、`Histogram`、`Summary`：单条指标。
- `CounterVec`、`GaugeVec`、`HistogramVec`：带 label 的指标集合。
- `Registry`：指标注册表。
- `Collector`：向 Registry 收集指标的接口。
- `promhttp.HandlerFor`：把 Registry 转成 HTTP handler。

## 2. 使用独立 Registry

默认 Registry 会注册 Go process 和 build info 指标。应用通常可以使用默认 Registry；大型服务或测试场景建议显式创建 Registry：

```go
type Metrics struct {
	Requests  *prometheus.CounterVec
	Duration  *prometheus.HistogramVec
	InFlight  *prometheus.GaugeVec
	Registry  *prometheus.Registry
}

func NewMetrics() *Metrics {
	reg := prometheus.NewRegistry()
	m := &Metrics{
		Requests: prometheus.NewCounterVec(
			prometheus.CounterOpts{
				Namespace: "order_service",
				Subsystem: "http",
				Name:      "requests_total",
				Help:      "Total HTTP requests handled.",
			},
			[]string{"method", "route", "status"},
		),
		Duration: prometheus.NewHistogramVec(
			prometheus.HistogramOpts{
				Namespace: "order_service",
				Subsystem: "http",
				Name:      "request_duration_seconds",
				Help:      "HTTP request duration in seconds.",
				Buckets:   prometheus.DefBuckets,
			},
			[]string{"method", "route"},
		),
		InFlight: prometheus.NewGaugeVec(
			prometheus.GaugeOpts{
				Namespace: "order_service",
				Subsystem: "http",
				Name:      "in_flight_requests",
				Help:      "Current number of HTTP requests in flight.",
			},
			[]string{"route"},
		),
		Registry: prometheus.NewRegistry(),
	}
	m.Registry.MustRegister(m.Requests, m.Duration, m.InFlight)
	return m
}
```

测试独立 Registry 可以避免全局注册状态污染。不要在每个请求里创建 Registry 或注册指标。

## 3. 暴露 metrics

```go
metrics := NewMetrics()

mux := http.NewServeMux()
mux.Handle("/metrics", promhttp.HandlerFor(
	metrics.Registry,
	promhttp.HandlerOpts{
		EnableOpenMetrics: true,
	},
))
```

建议把业务端口和 metrics 端口分开，尤其是 metrics 包含内部资源信息时：

```go
go func() {
	metricsMux := http.NewServeMux()
	metricsMux.Handle("/metrics", promhttp.HandlerFor(metrics.Registry, promhttp.HandlerOpts{}))
	_ = http.ListenAndServe("127.0.0.1:9091", metricsMux)
}()
```

在 Kubernetes 中，如果 Prometheus 需要跨 Pod 访问，就监听 Pod 网络可达的地址，同时使用 NetworkPolicy、Service 或认证限制访问。

## 4. 观察请求

一个通用观察流程：

```go
start := time.Now()
status := "200"
metrics.InFlight.WithLabelValues(route).Inc()
defer metrics.InFlight.WithLabelValues(route).Dec()

next.ServeHTTP(rw, r)

metrics.Requests.WithLabelValues(r.Method, route, status).Inc()
metrics.Duration.WithLabelValues(r.Method, route).
	Observe(time.Since(start).Seconds())
```

真实实现必须解决：

- 如何读取最终 status code。
- handler panic 时如何计入 500。
- 路由模板从哪里取得。
- 重定向和没有显式 `WriteHeader` 时状态码是什么。
- response writer 的 Flush、Hijack、Push 接口是否需要保留。
- middleware 是否被重复包裹。

## 5. 记录最终状态码

```go
type statusWriter struct {
	http.ResponseWriter
	status      int
	wroteHeader bool
}

func (w *statusWriter) WriteHeader(code int) {
	if !w.wroteHeader {
		w.status = code
		w.wroteHeader = true
	}
	w.ResponseWriter.WriteHeader(code)
}

func (w *statusWriter) Write(body []byte) (int, error) {
	if !w.wroteHeader {
		w.WriteHeader(http.StatusOK)
	}
	return w.ResponseWriter.Write(body)
}
```

生产项目中可以使用经过验证的 instrumentation middleware，但仍然要理解它怎样取得 route 和 status。若使用框架，优先使用框架的路由模板 API，不要从 `r.URL.Path` 推断模板。

## 6. 避免指标重复

常见重复来源：

- 网关统计一次，应用又统计一次，却在同一个 Dashboard 相加。
- middleware 同时包裹父路由和子路由。
- 重试逻辑把一次用户操作统计成多次依赖请求；这可能是正确的，但必须命名清楚。
- `/metrics` 自己也被通用 middleware 统计，污染业务请求量。

为每个指标写“统计边界”：

```text
http_requests_total：应用 handler 收到的用户请求，一次重试仍是一次请求。
dependency_requests_total：每次实际发出的下游 HTTP 请求，包括重试。
```

## 7. 练习

- 为 `/hello`、`/orders`、`/healthz` 记录不同 route。
- 让 `/orders` 产生 400、500、503，检查 status label。
- 用并发请求观察 in-flight 是否先升后降。
- 将 metrics handler 排除在业务 route 指标之外。
- 使用独立 Registry 编写测试，测试后能重复运行。

