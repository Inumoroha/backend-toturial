# 服务发现与 relabeling

## 1. 为什么需要服务发现

静态配置适合实验和少量稳定服务。生产中实例会因为发布、扩缩容和节点变化而变化，需要从平台获取 target，再用 relabel 生成稳定的 Prometheus labels。

常见方式：

- `file_sd_configs`：由部署系统生成 JSON 文件。
- DNS SD：服务地址由 DNS 返回。
- Kubernetes SD：从 API Server 发现 Pod、Service、Endpoints。
- Consul 等注册中心：从服务目录发现。

## 2. file_sd 示例

文件：

```json
[
  {
    "targets": ["app-1:8080", "app-2:8080"],
    "labels": {
      "job": "order-api",
      "environment": "staging"
    }
  }
]
```

配置：

```yaml
scrape_configs:
  - job_name: file-discovered
    file_sd_configs:
      - files:
          - /etc/prometheus/file_sd/*.json
        refresh_interval: 30s
```

文件由外部系统原子替换，避免 Prometheus 读到半写入内容。

## 3. target relabel

```yaml
relabel_configs:
  - source_labels: [__address__]
    regex: "([^:]+):\\d+"
    target_label: instance
    replacement: "$1"

  - source_labels: [__meta_kubernetes_namespace]
    target_label: namespace

  - source_labels: [__meta_kubernetes_pod_label_app]
    target_label: app

  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: "true"
```

`__meta_*` 是服务发现元数据，`__address__` 是抓取地址。以 `__` 开头的临时 label 在最终样本中通常不会保留。

## 4. 丢弃 target 和丢弃样本

丢弃 target：

```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_phase]
    action: keep
    regex: Running
```

丢弃样本：

```yaml
metric_relabel_configs:
  - source_labels: [__name__]
    action: drop
    regex: "debug_.*"
```

区别：

- target relabel 在抓取前执行，节省网络和解析。
- metric relabel 在抓取后执行，已经消耗抓取成本。
- metric relabel 不应被用来掩盖应用高基数问题。

## 5. relabel 调试方法

1. 查看 `/service-discovery` 页面（若版本和权限提供）。
2. 查看 `/targets` 中的 Discovered labels 和 Labels。
3. 暂时移除 keep/drop 规则，确认 target 原本能被发现。
4. 每次只改一条 relabel。
5. 用清晰的 regex，避免过度捕获。
6. 记录规则作用于 target 还是样本。

## 6. 常见错误

- selector 只匹配 Service，但实际需要 Pod。
- label 名称拼写或大小写不一致。
- `__meta_kubernetes_pod_annotation_*` 对应 annotation 不存在。
- rewrite `__address__` 后端口错误。
- keep 规则匹配空值，导致所有 target 被丢弃。
- metric relabel 后只剩下少量指标，却没有监控丢弃量。

## 7. 阶段练习

用 file_sd 完成：

- 2 个 Go API target。
- 1 个 exporter target。
- `environment` 和 `team` 标签。
- 一个只保留 `Running` 的规则。
- 一个丢弃 debug 指标的 metric relabel。

然后改变地址，确认 Prometheus 无需重启就能发现文件变化。

