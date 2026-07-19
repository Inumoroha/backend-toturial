# 阶段 6：Exporter、服务发现与 Kubernetes

当应用和基础设施数量增加后，静态 target 会变得难以维护。本阶段学习如何复用成熟 Exporter、编写小型 Exporter，并让 Prometheus 自动发现 Kubernetes 工作负载。

## 学习目标

- 理解 Exporter 的采集边界、错误语义和权限。
- 写一个读取 HTTP JSON 的 Go Exporter。
- 区分 target relabel 和 metric relabel。
- 使用 file_sd、DNS 和 Kubernetes SD。
- 掌握 ServiceMonitor、PodMonitor、RBAC、端口名和标签选择器。
- 排查“资源存在但没有被抓取”。

## 教程顺序

1. [编写一个可靠的 Go Exporter](01-writing-go-exporter.md)
2. [服务发现与 relabeling](02-service-discovery-relabeling.md)
3. [Kubernetes 监控](03-kubernetes-monitoring.md)

## 阶段项目

实现一个 `inventory-exporter`：

- 从模拟库存 HTTP API 读取商品库存。
- 暴露库存数量、请求成功、采集耗时和错误信息。
- 采集失败时返回 exporter 自身指标，并保留最后一次成功时间。
- 不把商品 ID 无限制地作为 label；使用有限商品类别或聚合维度。
- 在 kind/minikube 中部署并自动发现。

## 阶段交付物

- Exporter 设计文档和测试。
- 一份带 relabel 的 file_sd 或 Kubernetes 配置。
- 一个 ServiceMonitor/PodMonitor。
- 一份“未被发现”排障清单。

