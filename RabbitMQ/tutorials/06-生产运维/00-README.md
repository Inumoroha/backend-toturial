# 第 6 阶段：生产运维

> 本阶段目标：从“能开发 RabbitMQ 业务”升级到“能运维和排查 RabbitMQ 生产系统”。你要掌握管理后台、核心指标、Prometheus/Grafana、告警、权限、安全、容量规划、备份、集群基础和故障处理手册。

## 学习顺序

请按下面顺序学习：

1. [01-本阶段目标与生产运维思维.md](./01-本阶段目标与生产运维思维.md)
2. [02-Management-UI与诊断命令.md](./02-Management-UI与诊断命令.md)
3. [03-核心监控指标.md](./03-核心监控指标.md)
4. [04-Prometheus与Grafana监控.md](./04-Prometheus与Grafana监控.md)
5. [05-告警规则设计.md](./05-告警规则设计.md)
6. [06-容量规划与资源水位.md](./06-容量规划与资源水位.md)
7. [07-vhost用户权限与最小权限.md](./07-vhost用户权限与最小权限.md)
8. [08-TLS网络与安全加固.md](./08-TLS网络与安全加固.md)
9. [09-生产部署检查清单.md](./09-生产部署检查清单.md)
10. [10-配置备份与拓扑治理.md](./10-配置备份与拓扑治理.md)
11. [11-集群基础与Quorum-Queue入门.md](./11-集群基础与Quorum-Queue入门.md)
12. [12-线上故障排查手册.md](./12-线上故障排查手册.md)
13. [13-第6阶段练习与自测.md](./13-第6阶段练习与自测.md)

## 建议学习时间

建议用 10 到 14 天完成：

- 第 1 天：建立生产运维思维。
- 第 2 天：熟悉 Management UI 和诊断命令。
- 第 3 到 4 天：学习核心指标和 Prometheus/Grafana。
- 第 5 天：设计告警规则。
- 第 6 天：学习容量规划和资源水位。
- 第 7 天：学习 vhost、用户、权限和最小权限。
- 第 8 天：学习 TLS、网络和安全加固。
- 第 9 天：整理生产部署检查清单。
- 第 10 天：学习配置备份和拓扑治理。
- 第 11 到 12 天：学习集群基础和 quorum queue。
- 第 13 到 14 天：完成故障演练和自测。

## 本阶段你要掌握什么

完成本阶段后，你应该能做到：

- 使用 Management UI 判断 RabbitMQ 当前健康状态。
- 使用 `rabbitmqctl` 和 `rabbitmq-diagnostics` 做基础排查。
- 解释 ready、unacked、publish rate、deliver rate、ack rate。
- 使用 Prometheus 和 Grafana 监控 RabbitMQ。
- 设计队列堆积、消费者消失、内存、磁盘、连接数等告警。
- 规划连接数、channel 数、队列数量、磁盘和内存水位。
- 使用 vhost、user、permission 做权限隔离。
- 理解 TLS 和网络安全基本配置方向。
- 制定生产部署前检查清单。
- 备份 definitions，治理 exchange、queue、binding。
- 理解集群和 quorum queue 的基础运维思路。
- 面对消息堆积、unacked 高、连接暴涨、磁盘告警时能排查。

## 本阶段重要提醒

RabbitMQ 生产运维的核心不是“看到 UI 是绿色的”，而是能持续回答：

```text
消息是否在按预期进入？
消息是否在按预期消费？
失败是否可恢复？
资源是否还有余量？
权限是否足够收敛？
故障发生时是否能定位和止血？
```

## 官方资料

- [RabbitMQ Monitoring](https://www.rabbitmq.com/docs/monitoring)
- [RabbitMQ Production Checklist](https://www.rabbitmq.com/docs/production-checklist)
- [RabbitMQ Management Plugin](https://www.rabbitmq.com/docs/management)
- [RabbitMQ Prometheus Plugin](https://www.rabbitmq.com/docs/prometheus)
- [RabbitMQ Access Control](https://www.rabbitmq.com/docs/access-control)
- [RabbitMQ TLS Support](https://www.rabbitmq.com/docs/ssl)
- [RabbitMQ Clustering](https://www.rabbitmq.com/docs/clustering)
- [RabbitMQ Quorum Queues](https://www.rabbitmq.com/docs/quorum-queues)

