# 第 4 阶段：Consumer 消费者机制总览

这一阶段学习 Kafka consumer。Consumer 是 Go 后端项目里最容易出事故的部分，因为它涉及 offset 提交、业务处理、错误分类、重试、死信、重平衡和优雅退出。

---

## 一、本阶段课程

1. `1_Consumer拉取模型.md`
2. `2_ConsumerGroup与分区分配.md`
3. `3_自动提交与手动提交Offset.md`
4. `4_Rebalance机制与Go服务退出.md`
5. `5_ConsumerLag观察与排查.md`
6. `6_综合实践_手动提交与失败处理.md`

---

## 二、本阶段学习目标

完成后你应该能做到：

- 解释 consumer group 如何分配 partition。
- 解释自动提交 offset 的风险。
- 设计业务成功后再提交 offset 的流程。
- 理解 rebalance 发生的原因。
- 知道 consumer 退出时为什么要优雅关闭。
- 根据 lag 判断 consumer 是否落后。

---

## 三、本阶段 Go 后端视角

一个生产级 consumer 至少要考虑：

```text
context cancellation
manual commit
handler timeout
retry topic
DLQ
idempotency
structured logging
metrics
graceful shutdown
```

本阶段会先把 consumer 机制讲清楚，第 5 阶段再写 Go 代码。

---

## 四、本阶段最终产出

学完本阶段，你应该能设计一条可靠消费流程：

```text
poll message
-> handler 解析和处理业务
-> 成功：commit offset
-> 可重试失败：写 retry topic，成功后 commit offset
-> 不可重试失败：写 DLQ，成功后 commit offset
-> retry/DLQ 写失败：不 commit offset
```

这条流程是后续 Go consumer 封装的基础。

你还应该能看懂 consumer group 的状态：

```text
GROUP              TOPIC          PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
inventory-service  order.created  0          120             130             10
```

并解释 lag 为什么增长、为什么下降、为什么某个 partition 可能单独落后。

---

## 五、最小实验闭环

本阶段至少要完成一次手动实验：

```text
启动 consumer group
-> 消费 order.created
-> 查看 offset 前进
-> 停止 consumer
-> producer 继续发送消息
-> 查看 lag 增长
-> 重启 consumer
-> 查看 lag 下降
```

再做一次失败实验：

```text
handler 故意返回错误
-> 不提交 offset
-> 重启 consumer
-> 观察同一条消息再次被消费
```

这个实验能让你真正理解“至少一次”和“重复消费”。

---

## 六、Consumer 设计原则

### 1. 业务成功后再提交 offset

提前提交 offset 会导致业务失败后 Kafka 不再投递这条消息。

### 2. 提交失败要接受重复

业务成功但 commit 失败时，消息可能重复消费。业务必须幂等。

### 3. Rebalance 是正常现象

服务扩容、缩容、超时、发布都会触发 rebalance。consumer 代码要能承受 rebalance 带来的短暂停顿和重复消费。

### 4. Lag 是健康指标，不是单纯数字

短时间 lag 升高不一定有问题，持续升高才需要排查。还要看是否集中在某个 partition。

---

## 七、Go 后端常见错误

### 1. 自动提交 offset 用在关键业务

自动提交简单，但可能在业务还没成功时就提交进度。订单、库存、支付这类业务更适合手动提交。

### 2. handler 没有超时

handler 卡住会导致 partition 长时间不前进，也可能触发 rebalance。

### 3. 错误被吞掉

如果 handler 失败后返回 `nil`，consumer 框架会提交 offset，消息就可能丢失。

### 4. 只看服务是否存活

consumer 进程活着，不代表它正常消费。必须观察 lag、错误率、处理耗时和 DLQ。

---

## 八、本阶段验收

完成本阶段后，你应该能回答：

1. consumer group 如何分配 partition？
2. 为什么 consumer 数超过 partition 数后不一定提升吞吐？
3. 自动提交 offset 有什么风险？
4. 手动提交 offset 的推荐时机是什么？
5. rebalance 为什么可能导致重复消费？
6. lag 持续升高应该按什么顺序排查？

---

## 九、本阶段推荐实验记录

建议建立 `consumer-experiments.md`：

```markdown
# Consumer 实验记录

## Consumer Group 分配

- topic partitions:
- consumer 数:
- 分配结果:

## Offset 提交

- 自动提交还是手动提交:
- handler 成功后 offset:
- handler 失败后 offset:

## Rebalance

- 触发方式:
- 消费是否暂停:
- 是否出现重复消费:

## Lag

- 最大 lag:
- lag 下降速度:
- 是否存在热点 partition:
```

这份记录会帮助你进入第 5 阶段写 Go consumer。

---

## 十、Consumer 可靠性检查表

设计 consumer 时逐项检查：

- [ ] group id 是否稳定。
- [ ] 是否关闭自动提交。
- [ ] 是否业务成功后提交 offset。
- [ ] handler 是否有 timeout。
- [ ] 错误是否分类。
- [ ] 可重试错误是否进入 retry。
- [ ] 不可重试错误是否进入 DLQ。
- [ ] commit 失败是否记录日志。
- [ ] 业务是否幂等。
- [ ] 是否支持优雅退出。

少掉其中任何一项，关键业务 consumer 都可能出事故。

---

## 十一、和第 5 阶段的关系

第 4 阶段讲机制，第 5 阶段写代码。

你在第 5 阶段封装 consumer group 时，实际就是把第 4 阶段的规则变成代码：

```text
FetchMessage
-> handler.Handle
-> classify error
-> publish retry/DLQ if needed
-> commit offset
```

如果第 4 阶段没理解清楚，第 5 阶段很容易写出“能跑但不可靠”的 consumer。

---

## 十二、阶段完成标准

你可以进入第 5 阶段的标志是：

- 能解释 group id 的作用。
- 能解释 partition 分配限制。
- 能说清自动提交为什么危险。
- 能写出手动提交流程。
- 能制造一次重复消费。
- 能通过 lag 判断 consumer 是否积压。

---

## 十三、推荐复盘问题

完成本阶段后，写下：

```text
如果 handler 成功但 commit offset 失败，会发生什么？
如果 handler 失败但代码返回 nil，会发生什么？
如果 consumer 数超过 partition 数，为什么吞吐不一定增加？
如果 lag 只集中在一个 partition，可能是什么原因？
```

这些问题是后续 Go consumer 封装必须处理的真实风险。

---

## 十四、下一阶段预告

第 5 阶段会把本阶段机制变成 Go 代码：

- 定义 consumer handler 接口。
- 封装 producer。
- 封装 consumer group。
- 处理 context 和优雅退出。
- 把 retry/DLQ 的边界放进代码结构。

如果第 4 阶段理解扎实，第 5 阶段就不是“照抄代码”，而是把规则工程化。

---

## 十五、阶段结束小作业

设计 `inventory-service` 的消费策略：

```text
group id:
订阅 topic:
是否自动提交:
handler 成功后:
handler 可重试失败:
handler 不可重试失败:
rebalance 时:
lag 告警:
```

写完后检查：是否能避免消息静默丢失，是否能承受重复消费。

---

## 十六、学习产物检查

本阶段结束时，应该留下：

- consumer group 分区分配实验记录。
- 自动提交和手动提交对比笔记。
- 一次重复消费实验记录。
- 一次 lag 增长和下降截图或文字记录。
- 一份可靠消费伪代码。

这些产物会直接指导第 5 阶段的 Go consumer 封装。

---

## 十七、面试表达

可以这样描述 consumer 可靠性：

```text
关键业务 consumer 我会关闭自动提交，等业务处理成功后再提交 offset。
如果业务失败，会先按错误类型写 retry 或 DLQ，写成功后才提交原 offset。
由于 commit 失败、rebalance 等场景仍可能重复消费，所以业务层必须通过 event_id 或状态机做幂等。
```

---

## 十八、推荐学习时间安排

| 天数 | 任务 | 重点 |
| --- | --- | --- |
| 第 1 天 | Consumer 拉取模型 | poll/fetch、批量拉取 |
| 第 2 天 | ConsumerGroup | 分区分配、group id |
| 第 3 天 | Offset 提交 | 自动提交、手动提交 |
| 第 4 天 | Rebalance 和退出 | 重复消费、优雅退出 |
| 第 5 天 | Lag 排查和综合实践 | lag、DLQ、失败处理 |

每天都要做一个小实验。Consumer 的坑不在 API，而在失败时的行为。

---

## 十九、代码化前的伪代码

进入第 5 阶段写 Go 之前，先能写出：

```text
while running:
  msg = fetch()
  err = handler(msg)
  if err == nil:
    commit(msg)
  else if retryable:
    publish retry
    commit(msg)
  else:
    publish dlq
    commit(msg)
```

并且补上一句：

```text
publish retry 或 dlq 失败时，不 commit。
```

---

## 二十、最终检查

进入 Go 编码前，先口头解释一次：

```text
同一条消息从 fetch 到 handler 到 commit 的完整生命周期。
```

如果解释时把 fetch 成功和业务成功混在一起，建议重读 offset 章节。

---

## 二十一、下一步连接

第 4 阶段讲的是 consumer 机制。第 5 阶段会把这些机制写成 Go 封装，重点是 handler 接口、context、日志和 offset 提交边界。

---

## 二十二、复习建议

每次写 consumer 代码前，先问三句：

```text
业务成功了吗？
offset 提交了吗？
重复消费安全吗？
```

这三句能避免大多数初学者常犯的 consumer 可靠性错误。

这就是可靠 consumer 的核心骨架。

---

## 二十三、故障演练

本阶段最后做一次：

```text
handler 成功后，commit 前让进程退出。
```

重启后观察是否重复消费。这个实验能把 offset 提交时机真正刻进脑子里。

---

## 二十四、最终产物

本阶段建议留下：

```text
consumer-group-notes.md
offset-commit-notes.md
lag-playbook.md
rebalance-notes.md
```

这些笔记后面会变成 Go consumer 封装的规则。

---

## 二十二、本阶段验收标准

Consumer 阶段结束后，你应该能把一次消费写成完整流程：

```text
poll 拉取消息。
反序列化并校验事件。
执行业务 handler。
业务成功后提交 offset。
业务失败时进入 retry 或 DLQ。
进程退出时停止拉取，等待正在处理的消息结束。
rebalance 时释放旧分区。
```

如果这段流程里任何一步说不清，Go 代码里就容易出现隐藏 bug，例如提前提交 offset、退出时丢消息、rebalance 后重复处理。
