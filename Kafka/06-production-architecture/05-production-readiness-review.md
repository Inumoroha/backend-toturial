# 05 Kafka 链路上线评审模板

本节目标：在 Kafka 链路上线前，用一份模板检查 topic、producer、consumer、可靠性、监控、安全和回滚方案。

## 1. 业务背景

需要写清楚：

- 这个 Kafka 链路解决什么业务问题？
- 是否是关键链路？
- 消息丢失的业务影响是什么？
- 重复消费的业务影响是什么？
- 可接受延迟是多少？

示例：

```text
order-service 发布 order.created，inventory-service 消费后扣减库存。
这是关键链路。
消息丢失会导致订单创建但库存未扣减。
重复消费可能导致库存重复扣减，因此必须幂等。
可接受端到端延迟：5 秒内。
```

## 2. Topic 设计

| 项 | 内容 |
| --- | --- |
| topic 名称 | `order.created` |
| partition 数 | 12 |
| replication factor | 3 |
| retention | 7 天 |
| cleanup policy | delete |
| message key | `order_id` |
| schema 格式 | JSON v1 或 Protobuf |

需要回答：

- partition 数量如何估算？
- 是否需要保证同一 key 顺序？
- retention 是否覆盖最长恢复时间？
- 是否有 retry 和 DLQ topic？

## 3. Producer 评审

检查项：

- 是否统一 producer 封装。
- 是否设置发送超时。
- 是否有发送失败处理。
- 是否设置可靠性参数。
- 是否有日志和指标。
- 服务退出前是否 flush 或 close。
- 是否使用 outbox pattern。

关键业务推荐：

```text
acks=all
enable.idempotence=true
retries 合理开启
message key 稳定
```

## 4. Consumer 评审

检查项：

- 是否关闭自动提交。
- 是否业务成功后提交 offset。
- 是否支持幂等。
- 是否有 retry topic。
- 是否有 DLQ。
- 是否能优雅退出。
- 是否有超时控制。
- 是否有最大重试次数。

需要明确：

```text
哪些错误本地重试？
哪些错误进入 retry topic？
哪些错误进入 DLQ？
写 retry 或 DLQ 失败怎么办？
```

## 5. 幂等与一致性

必须写清楚：

- 幂等键是什么？
- 去重表在哪里？
- 哪些数据库操作在同一个事务中？
- 外部接口是否支持幂等键？
- DLQ 重放是否安全？

示例：

```text
inventory-service 使用 event_id 写 processed_events。
processed_events 插入和库存扣减在同一个数据库事务内。
重复 event_id 直接返回成功并提交 offset。
```

## 6. 可观测性

指标：

- producer success/error count。
- consumer success/error count。
- handler duration P95/P99。
- consumer lag。
- retry count。
- DLQ count。

日志字段：

- trace_id。
- event_id。
- topic。
- partition。
- offset。
- key。
- group。
- attempt。

告警：

- lag 持续升高。
- DLQ 有新增。
- 错误率升高。
- handler P99 超阈值。

## 7. 安全

检查：

- 是否启用 TLS。
- 是否启用 SASL。
- 是否按服务配置 ACL。
- 是否避免多个服务共享同一账号。
- 是否避免在消息中放敏感字段。

## 8. 回滚与应急

上线前必须知道：

- producer 是否可以临时关闭。
- consumer 是否可以暂停。
- 是否可以重放 DLQ。
- 是否可以从某个 offset 恢复。
- topic 配置改错如何回退。
- 消息格式变更是否兼容旧 consumer。

## 9. 最终批准问题

上线前逐项回答：

1. 如果 Kafka 暂时不可用，业务如何处理？
2. 如果 consumer 处理成功但提交 offset 失败，会怎样？
3. 如果消息重复 10 次，业务结果是否正确？
4. 如果某条消息永远处理失败，会不会卡住整个 partition？
5. 如果消息格式升级，旧 consumer 是否还能工作？
6. 如果 lag 持续升高，第一步看什么？

## 本节练习

为项目实战中的 `order.created -> inventory-service` 链路填写一份上线评审。

