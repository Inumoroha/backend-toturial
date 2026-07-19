# HA、远程存储与安全

## 1. 高可用不是简单复制

两个 Prometheus 实例可以同时抓取同一批目标，但会带来：

- 两份本地数据。
- 两次规则评估。
- 可能重复告警。
- 查询结果需要去重或选择来源。
- 外部 labels 必须区分集群、区域和副本。

Prometheus 本身不自动把两个独立实例合成一个一致的数据库。HA 方案要明确：

- 抓取副本。
- 告警去重。
- 查询入口。
- 远程存储去重。
- 失联期间如何恢复。

## 2. external_labels

```yaml
global:
  external_labels:
    cluster: prod-a
    replica: prometheus-0
```

external label 用于远程存储和告警去重。不要把可变的 Pod ID 当作业务维度，也不要让两个副本使用完全相同的 identity。

## 3. Remote Write

远程写入适合：

- 长期保留。
- 多集群集中查询。
- 本地 Prometheus 磁盘受限。
- 跨区域汇总。

监控：

```promql
rate(prometheus_remote_storage_samples_total[5m])
rate(prometheus_remote_storage_samples_failed_total[5m])
prometheus_remote_storage_pending_samples
prometheus_remote_storage_shards
```

一个最小的配置形态如下，实际后端地址、认证和 TLS 参数必须按所选长期存储系统填写：

```yaml
remote_write:
  - url: https://metrics-store.example/api/v1/write
    timeout: 30s
    queue_config:
      capacity: 10000
      max_shards: 20
      min_shards: 1
      max_samples_per_send: 2000
      batch_send_deadline: 5s
    tls_config:
      ca_file: /etc/prometheus/certs/ca.crt
    headers:
      X-Scope-OrgID: platform
```

`queue_config` 的数值要结合 samples/s、网络带宽和后端吞吐压测；队列越大不代表故障期间可以无限缓存。写入端点的认证 header、证书和租户标识必须通过 Secret 或安全配置注入，不能把真实凭据提交到 Git。

关注失败、队列积压、重试和恢复后的追赶时间。远程写入不是备份的替代品，要确认后端持久化、权限、容量和恢复流程。

## 4. Federation 和长期系统

联邦适合抓取已聚合的上层指标，减少跨层查询数据量。长期系统（如 Thanos、Mimir、Cortex、VictoriaMetrics 等）提供不同的存储、查询、租户和运维取舍。

选择方案时比较：

- 数据保留和成本。
- 多租户和权限。
- 查询延迟。
- HA 去重。
- 运维复杂度。
- 与现有 Grafana、告警和备份的集成。
- 数据迁移和退出成本。

不要先选产品，再倒推需求。

## 5. 安全边界

Prometheus、Alertmanager 和 Grafana 都是运维系统，不应直接暴露公网：

- API 使用身份认证和网络访问控制。
- metrics endpoint 只对 Prometheus 开放。
- remote write/read 使用 TLS 和最小权限。
- Alertmanager webhook 做认证、超时和重试。
- Grafana 管理员密码使用 Secret，不写进 Git。
- 查询中避免把敏感数据放进 label。
- 配置变更和 silence 操作有审计记录。

## 6. 备份和恢复

明确备份对象：

- Prometheus 配置和规则。
- Alertmanager 路由和静默策略（按产品能力）。
- Grafana provisioning、Dashboard 和数据源配置。
- 长期存储后端数据。
- 运行手册和版本信息。

本地 TSDB 的快照不是完整的 HA 方案；需要定期做恢复演练，测量 RPO 和 RTO。

## 7. 阶段验收

画出两个 Prometheus 副本、Alertmanager、Grafana 和长期存储的关系图，并写出任一组件故障时：

- 用户能否继续查询。
- 告警是否重复或丢失。
- 数据是否最终补齐。
- 谁负责手工介入。

补充一张故障矩阵：

| 故障 | 查询 | 新数据 | 告警 | 恢复证据 |
| --- | --- | --- | --- | --- |
| 单个 Prometheus 宕机 | 是否由另一副本提供 | 是否有采集缺口 | 是否重复/丢失 | target、remote write、Alertmanager 状态 |
| Remote Write 后端不可用 | 本地近期数据是否可查 | 队列是否积压 | 是否有写入失败告警 | pending samples 回落 |
| Alertmanager 不可用 | Grafana 是否正常 | Prometheus 是否继续评估 | 通知是否延迟 | Resolved/通知日志 |
| Grafana 不可用 | Prometheus API 是否可查 | 不受影响 | 不受影响 | Dashboard 恢复 |
