# 5. Kafka 链路上线评审

本节目标：上线 Kafka 链路前，用一份清单检查 topic、producer、consumer、可靠性、监控、安全和回滚。

---

## 一、业务背景

必须写清：

- 链路解决什么问题？
- 是否关键链路？
- 消息丢失影响？
- 重复消费影响？
- 可接受延迟？

示例：

```text
order.created 用于通知库存扣减。
消息丢失会导致订单创建但库存不扣。
重复消费可能重复扣库存。
要求 5 秒内完成。
```

---

## 二、Topic 评审

| 项 | 内容 |
| --- | --- |
| topic | order.created |
| partitions | 12 |
| replication factor | 3 |
| retention | 7 天 |
| key | order_id |
| schema | JSON v1 |

还要有 retry 和 DLQ。

---

## 三、Producer 评审

检查：

- 是否统一封装。
- 是否设置 key。
- 是否有 event_id。
- 是否处理发送失败。
- 是否使用 outbox。
- 是否有超时。
- 是否 close/flush。

---

## 四、Consumer 评审

检查：

- 是否手动提交 offset。
- 是否业务成功后提交。
- 是否幂等。
- 是否支持 retry。
- 是否支持 DLQ。
- 是否优雅退出。
- 是否有超时。

---

## 五、可靠性评审

必须回答：

- event_id 如何生成？
- 去重表在哪里？
- 哪些操作在同一事务？
- retry 最大次数？
- DLQ 如何重放？
- commit 失败怎么办？

---

## 六、可观测性评审

指标：

- producer 成功/失败。
- consumer 成功/失败。
- handler P95/P99。
- lag。
- retry。
- DLQ。

日志：

```text
topic partition offset key event_id group handler
```

---

## 七、安全评审

检查：

- TLS。
- SASL。
- ACL。
- 是否最小权限。
- 是否有敏感字段。

---

## 八、回滚方案

必须知道：

- producer 是否可暂停。
- consumer 是否可暂停。
- offset 是否可调整。
- DLQ 是否可重放。
- schema 是否兼容。

---

## 九、本节练习

为 `order.created -> inventory-service` 填写一份上线评审。

---

## 十、本节小结

- Kafka 链路上线前必须评审。
- 评审要覆盖 topic、producer、consumer、可靠性、监控、安全、回滚。
- 关键问题是消息丢失、重复消费和恢复能力。

---

## 十一、完整评审模板

```markdown
# Kafka 链路上线评审

## 业务背景

## Topic 设计

## Message Key

## Schema 示例

## Producer 设计

## Consumer 设计

## 幂等设计

## Retry/DLQ 设计

## Outbox 设计

## 监控告警

## 安全 ACL

## 容量估算

## 发布计划

## 回滚方案

## 风险和待办
```

---

## 十二、order.created 评审示例

业务背景：

```text
订单创建后发布 order.created，库存、通知、分析服务分别消费。
```

Topic：

```text
order.created
partitions=12
replication.factor=3
retention=7d
key=order_id
```

Producer：

```text
order-service 写 outbox_events
outbox-worker 发布 Kafka
acks=all
enable.idempotence=true
```

Consumer：

```text
inventory-service group=inventory-service
manual commit
processed_events 幂等
retry/DLQ
```

---

## 十三、风险登记表示例

| 风险 | 影响 | 应对 |
| --- | --- | --- |
| Kafka 短暂不可用 | outbox 积压 | worker 重试，监控 PENDING age |
| 库存服务宕机 | lag 增长 | 恢复后补消费，retention 7d |
| 消息格式错误 | 消费失败 | DLQ + 告警 |
| 重复消息 | 重复扣库存 | processed_events 幂等 |
| 热点 key | 单 partition lag 高 | 监控 partition lag，评估 key 拆分 |

---

## 十四、发布计划示例

```text
1. 创建 topic、retry、DLQ。
2. 配置 ACL。
3. 部署 consumer，先不处理或只记录。
4. 部署 outbox worker。
5. 打开 order-service outbox 写入。
6. 小流量验证。
7. 观察 lag、DLQ、错误率。
8. 全量发布。
```

---

## 十五、回滚计划示例

```text
关闭 producer 开关，停止新事件写入。
consumer 可以继续处理已写入事件。
如 consumer 有 bug，先暂停 consumer，修复后从原 offset 继续。
如消息格式有问题，进入 DLQ，修复后重放。
```

回滚不是简单“回滚代码”，还要考虑 Kafka 中已经存在的消息。

---

## 十六、评审通过标准

- [ ] topic 已创建并确认配置。
- [ ] ACL 已配置。
- [ ] producer 失败有处理。
- [ ] consumer 手动提交 offset。
- [ ] 幂等方案已验证。
- [ ] retry/DLQ 已验证。
- [ ] lag 和 DLQ 有告警。
- [ ] schema 兼容。
- [ ] 有回滚方案。

---

## 十七、评审会议怎么开

上线评审不是一个人填表就结束。建议至少包含：

```text
业务负责人
producer 服务负责人
consumer 服务负责人
DBA 或基础设施负责人
值班或运维负责人
```

会议顺序：

1. 说明业务链路和影响范围。
2. 过 topic、schema、key 和容量估算。
3. 过 producer 失败处理。
4. 过 consumer 幂等、retry、DLQ。
5. 过监控告警和排障命令。
6. 过发布和回滚计划。
7. 明确遗留风险和负责人。

如果一个问题没人负责，就不能算评审通过。

---

## 十八、必须现场演示的内容

关键链路上线前，最好现场演示：

```text
正常消息能被消费。
重复消息不会重复产生副作用。
坏消息进入 DLQ。
consumer 停止后 lag 增长。
consumer 恢复后 lag 下降。
Kafka 不可用时 producer 或 outbox 有明确失败处理。
```

演示命令示例：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server prod-kafka:9092 \
  --describe \
  --group inventory-service
```

DLQ 查看：

```bash
kafka-console-consumer.sh \
  --bootstrap-server prod-kafka:9092 \
  --topic order.created.dlq \
  --from-beginning \
  --max-messages 1
```

生产环境不一定允许直接用 console 命令，但你至少要有等价的内部工具。

---

## 十九、上线前风险分级

| 风险等级 | 示例 | 要求 |
| --- | --- | --- |
| P0 | 消息丢失导致资金或库存错误 | 必须阻断上线 |
| P1 | DLQ 无告警 | 上线前修复 |
| P2 | 指标不完整但日志可查 | 记录待办和完成时间 |
| P3 | README 排障命令不够细 | 不阻断，但要补 |

Kafka 链路一旦上线，错误往往不是立刻显现，而是在 lag、DLQ、重复消费里慢慢积累。所以评审要偏严格。

---

## 二十、Go 后端评审重点

对 Go 服务要特别看：

- `context timeout` 是否设置。
- goroutine 是否能退出。
- producer close/flush 是否调用。
- consumer 是否处理 rebalance。
- 数据库事务是否覆盖幂等记录和业务操作。
- 错误是否被分类，而不是全部 `return nil`。
- 日志是否包含 event_id。

代码评审时看到下面写法要警惕：

```go
if err != nil {
    log.Println(err)
    return nil
}
```

这会让 consumer 框架以为处理成功，然后提交 offset。

---

## 二十一、评审结论模板

```markdown
## 评审结论

结论：通过 / 有条件通过 / 不通过

阻断项：
- 

上线前必须完成：
- 

上线后观察 24 小时：
- consumer lag
- DLQ 新增
- producer error rate
- outbox oldest pending age

负责人：
- producer：
- consumer：
- 运维：
```

这比一句“评审通过”更有工程价值。

---

## 二十一、评审结论写法

上线评审最后不要只写“通过”。

更好的写法是：

```text
结论：有条件通过。
必须完成项：补充 DLQ 重放权限控制。
风险接受项：首版暂不支持跨地域容灾。
上线窗口：2026-xx-xx 22:00-23:00。
观察指标：发送失败率、consumer lag、DLQ 数量、P99。
回滚触发条件：lag 持续 15 分钟不下降或 DLQ 快速增长。
```

这种结论能让上线动作有边界，也方便事后复盘。
