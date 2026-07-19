# Kubernetes 监控

## 1. 先理解对象关系

```text
Deployment -> Pod
Service -> Pod selector
Prometheus/Operator -> API Server / Service / Pod metadata
ServiceMonitor/PodMonitor -> 选择要抓取的 Service/Pod
```

Prometheus 抓取失败常常不是应用问题，而是：

- Service selector 没选中 Pod。
- Service 端口名和 ServiceMonitor 不一致。
- ServiceMonitor 所在 namespace 没被选择。
- Prometheus 没有读取目标 namespace 的 RBAC 权限。
- NetworkPolicy 阻断了 Prometheus 到 Pod 的访问。

## 2. Go 服务 Deployment 和 Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-api
  labels:
    app.kubernetes.io/name: order-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: order-api
  template:
    metadata:
      labels:
        app.kubernetes.io/name: order-api
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: /metrics
        prometheus.io/port: "9091"
    spec:
      containers:
        - name: app
          image: example/order-api:dev
          ports:
            - name: http
              containerPort: 8080
            - name: metrics
              containerPort: 9091
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
---
apiVersion: v1
kind: Service
metadata:
  name: order-api
  labels:
    app.kubernetes.io/name: order-api
spec:
  selector:
    app.kubernetes.io/name: order-api
  ports:
    - name: http
      port: 8080
      targetPort: http
    - name: metrics
      port: 9091
      targetPort: metrics
```

端口名要稳定。`targetPort: metrics` 依赖容器端口名存在。

## 3. ServiceMonitor 示例

使用 Prometheus Operator 时：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: order-api
  labels:
    team: backend
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: order-api
  endpoints:
    - port: metrics
      path: /metrics
      interval: 15s
      scrapeTimeout: 10s
  namespaceSelector:
    matchNames:
      - orders
```

这要求：

- Operator 安装 CRD。
- Prometheus 选择了带 `team: backend` 的 ServiceMonitor。
- Prometheus 的 RBAC 允许访问 `orders` namespace。
- ServiceMonitor selector 与 Service labels 匹配。
- endpoint `port` 是 Service 的端口名，不是容器端口号。

## 4. 原生 Kubernetes SD 思路

不使用 Operator 时，Prometheus 可以使用 `kubernetes_sd_configs`：

```yaml
scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: "(.+)"
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: "([^:]+)(?::\\d+)?;(\\d+)"
        replacement: "$1:$2"
        target_label: __address__
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
```

这类配置要求 Prometheus ServiceAccount 有读取 Pod、Service 和 Endpoint 等资源的权限。

## 5. RBAC 检查

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:monitoring:prometheus \
  -A

kubectl get servicemonitor -A
kubectl describe servicemonitor order-api -n orders
kubectl get endpoints order-api -n orders
kubectl get pods -n orders --show-labels
```

## 6. 发布和重启实验

1. 部署 2 个 Pod。
2. 查看 target 中的两个 instance。
3. 滚动更新镜像。
4. 观察旧 Pod 下线、新 Pod 上线和序列变化。
5. 删除一个 Pod，确认告警和恢复。
6. 修改 ServiceMonitor 端口名，观察 target 消失。
7. 修复端口名并确认自动恢复。

## 7. 安全边界

- Prometheus API Server 访问使用最小 RBAC。
- metrics endpoint 只允许 Prometheus 网络访问。
- 不把 Pod annotation 当成可信任的任意 URL 输入。
- NetworkPolicy 明确允许 Prometheus 到 metrics 端口。
- 生产镜像使用不可变 tag 和签名/扫描流程。

## 8. Kubernetes 基础设施指标的分工

应用 `/metrics` 只覆盖业务服务。集群监控通常还需要区分：

| 来源 | 主要内容 | 典型用途 |
| --- | --- | --- |
| kube-state-metrics | Deployment、Pod、Job、Node 对象状态 | 副本数、Pending、重启和期望/实际状态 |
| kubelet/cAdvisor | 容器 CPU、内存、网络和文件系统 | 容器资源使用和工作集 |
| node-exporter | 节点操作系统和硬件 | 节点 CPU、磁盘、文件描述符、网络 |
| 应用 `/metrics` | 请求、依赖和业务 | 用户影响和服务排障 |

避免重复抓取同一来源。先确认每个指标由哪个组件负责，再决定 job、instance、namespace、pod 和 node 的 label 关系。

常见查询示例：

```promql
sum by (namespace, pod) (rate(container_cpu_usage_seconds_total[5m]))
kube_pod_container_status_restarts_total > 0
node_filesystem_avail_bytes / node_filesystem_size_bytes
```

这些指标的 label 会随部署方式变化，实际查询前先用 `count by (...)` 查看当前环境的 label 集合。

## 9. 阶段验收

能够从以下证据定位 K8s 抓取失败：

- `kubectl get` 的 labels 和 ports。
- ServiceMonitor 的 selector。
- Prometheus `/targets` 的错误。
- Prometheus ServiceAccount 的 `auth can-i`。
- Pod 内部 `/metrics` 请求。
- NetworkPolicy 和 DNS 结果。
