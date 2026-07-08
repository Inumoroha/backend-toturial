# 10：Go 服务可观测性：Prometheus 与 OpenTelemetry

## 1. 本节目标

CI/CD 的最终目标是把服务安全、稳定地交付给用户。

所以第 9 阶段不能只看流水线本身，还要看发布后的 Go 服务。

这一节学习：

- Go 服务如何暴露 Prometheus metrics。
- 应该记录哪些应用指标。
- 如何控制 label cardinality。
- OpenTelemetry 在 trace 中解决什么问题。
- 如何把发布事件和运行时指标联系起来。

## 2. 最小 Prometheus metrics

安装：

```bash
go get github.com/prometheus/client_golang/prometheus
go get github.com/prometheus/client_golang/prometheus/promhttp
```

HTTP 服务增加 `/metrics`：

```go
package main

import (
	"log"
	"net/http"

	"github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {
	mux := http.NewServeMux()

	mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		_, _ = w.Write([]byte("ok"))
	})

	mux.Handle("/metrics", promhttp.Handler())

	log.Println("listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}
```

本地访问：

```bash
curl http://localhost:8080/metrics
```

## 3. 自定义业务指标

示例：统计 HTTP 请求数。

```go
var httpRequestsTotal = prometheus.NewCounterVec(
	prometheus.CounterOpts{
		Name: "go_cicd_lab_http_requests_total",
		Help: "Total number of HTTP requests.",
	},
	[]string{"method", "route", "status"},
)

func init() {
	prometheus.MustRegister(httpRequestsTotal)
}
```

在 handler 中记录：

```go
httpRequestsTotal.WithLabelValues("GET", "/healthz", "200").Inc()
```

更完整的项目会通过 middleware 自动记录：

- 请求数。
- 错误数。
- 请求耗时 histogram。
- in-flight 请求数。

## 4. 指标命名建议

好的指标名：

```text
go_cicd_lab_http_requests_total
go_cicd_lab_http_request_duration_seconds
go_cicd_lab_jobs_processed_total
go_cicd_lab_database_errors_total
```

原则：

- 用 `_total` 表示 counter。
- 时间单位写进名称，例如 `_seconds`。
- 名称描述业务含义。
- 不要把环境写进指标名，环境用 label 或采集配置区分。

## 5. 控制 label cardinality

危险：

```go
httpRequestsTotal.WithLabelValues("GET", r.URL.Path, "200").Inc()
```

如果路径包含用户 ID：

```text
/users/1
/users/2
/users/3
```

会产生大量 label value。

更好：

```text
route="/users/{id}"
```

避免作为 label：

- user id。
- email。
- request id。
- trace id。
- 原始 URL。
- 错误消息全文。

高 cardinality 会让 Prometheus 存储和查询变慢。

## 6. Go 服务四个黄金信号

发布后最常看：

```text
traffic：请求量。
errors：错误率。
latency：延迟。
saturation：资源饱和度。
```

PromQL 示例：

请求速率：

```promql
sum(rate(go_cicd_lab_http_requests_total[5m]))
```

错误率：

```promql
sum(rate(go_cicd_lab_http_requests_total{status=~"5.."}[5m]))
/
sum(rate(go_cicd_lab_http_requests_total[5m]))
```

p95 延迟：

```promql
histogram_quantile(
  0.95,
  sum(rate(go_cicd_lab_http_request_duration_seconds_bucket[5m])) by (le)
)
```

## 7. OpenTelemetry 解决什么问题

metrics 告诉你：

```text
错误率升高了。
```

trace 帮你追：

```text
一个请求经过 API、数据库、缓存、外部服务时，哪一步慢了或失败了。
```

OpenTelemetry 提供统一方式采集：

- traces。
- metrics。
- logs。

对 Go 后端而言，优先学习：

- HTTP server instrumentation。
- HTTP client instrumentation。
- database instrumentation。
- trace id 透传。

## 8. 发布信息进入运行时

让服务暴露版本信息：

```go
var (
	version = "dev"
	commit  = "unknown"
)

func versionHandler(w http.ResponseWriter, r *http.Request) {
	_, _ = w.Write([]byte("version=" + version + "\ncommit=" + commit + "\n"))
}
```

构建时注入：

```bash
go build \
  -ldflags "-X main.version=v1.2.3 -X main.commit=$GITHUB_SHA" \
  -o app ./cmd/api
```

发布后：

```bash
curl https://api.example.com/version
```

这能帮助你把运行中的服务和 CI/CD 发布证据关联起来。

## 9. Kubernetes ServiceMonitor 示例

如果使用 Prometheus Operator，可以配置：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: go-cicd-lab
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: go-cicd-lab
  namespaceSelector:
    matchNames:
      - go-cicd-lab-production
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```

具体字段取决于你的集群监控栈。

## 10. 小练习

完成：

1. 给 Go 服务增加 `/metrics`。
2. 增加一个请求计数器。
3. 增加 `/version`，输出 commit。
4. 构建镜像时注入 commit。
5. 发布后通过 `/version` 验证当前运行版本。
6. 写 3 条 PromQL：请求量、错误率、p95 延迟。

## 11. 本节小结

你现在应该理解：

- Go 服务应暴露 `/metrics`，便于发布后验证。
- 指标要控制 label cardinality。
- 四个黄金信号是 traffic、errors、latency、saturation。
- OpenTelemetry trace 帮你定位跨服务调用链路。
- 运行时版本信息能把服务和 CI/CD 发布记录关联起来。

