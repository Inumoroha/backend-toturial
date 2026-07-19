# 阶段 7：生产能力

本阶段把“本地能查到数据”提升为“生产监控系统可持续运行”。重点是数据量、可靠性、安全和变更。

## 学习目标

- 理解 TSDB、WAL、block、保留和容量估算。
- 设计远程写入、长期存储和 HA 方案。
- 监控 Prometheus 自身的抓取、查询、规则和存储健康。
- 优化高基数、慢查询和高频 Dashboard。
- 保护 Prometheus API、metrics、Alertmanager 和 Grafana。

## 教程顺序

1. [TSDB、保留和容量估算](01-tsdb-retention-capacity.md)
2. [HA、远程存储与安全](02-ha-remote-storage-security.md)
3. [性能、升级与日常运维](03-performance-upgrade-operations.md)

## 阶段交付物

- 一份按 target、series、scrape interval 和保留期计算的容量估算。
- 一份单副本、双副本和长期存储的方案比较。
- 一份 Prometheus 自监控 Dashboard 和告警列表。
- 一份升级、回滚、备份和恢复 Runbook。
- 一次高基数和一次慢查询演练。

## 阶段验收

能够回答：

- 现在的数据还能保留多久？
- 新增一个 label 会增加多少序列？
- Prometheus 自己变慢时谁发现？
- 单个 Prometheus 宕机时查询和告警有什么影响？
- 如何证明远程写入没有悄悄丢数据？

