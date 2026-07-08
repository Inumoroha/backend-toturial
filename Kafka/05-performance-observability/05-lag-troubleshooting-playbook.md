# 05 Consumer Lag 排查手册

本节目标：遇到 consumer lag 持续升高时，能按固定流程定位问题，而不是只会“加机器”。

## 先判断是不是问题

Lag 短时间升高不一定是事故。

需要告警的情况：

- lag 持续增长。
- lag 长时间不下降。
- 关键业务 topic lag 超过业务可接受时间。
- 单个 partition lag 异常高。
- lag 升高同时错误率升高。

## 第一步：确认现象

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group inventory-service
```

记录：

- 哪个 topic。
- 哪些 partition。
- `CURRENT-OFFSET`。
- `LOG-END-OFFSET`。
- `LAG`。
- consumer id。

如果只有一个 partition lag 高，优先怀疑热点 key 或坏消息。

如果所有 partition lag 都高，优先怀疑整体处理能力不足或下游依赖变慢。

## 第二步：看写入流量是否增加

问题：

- producer 是否突然写入更多消息？
- 是否有批量任务开始运行？
- 是否有上游重放历史数据？

如果写入突增，consumer lag 增长可能是正常排队。此时关注恢复时间是否可接受。

## 第三步：看 Consumer 错误率

检查 consumer 日志：

```text
handle failed
commit failed
json unmarshal failed
database timeout
remote service timeout
```

错误率高时，不要先扩容 consumer。先判断是：

- 消息格式问题。
- 数据库问题。
- 下游接口问题。
- 代码 bug。

## 第四步：看 Handler 耗时

指标：

```text
kafka_consumer_handle_duration_seconds
```

关注：

- P95。
- P99。
- 最大值。

如果平均耗时正常但 P99 很高，说明少量慢消息可能拖住 partition。

## 第五步：看 Rebalance

频繁 rebalance 会导致消费不稳定。

常见日志关键词：

```text
rebalance
join group
sync group
heartbeat failed
session timeout
max poll interval exceeded
```

可能原因：

- handler 执行太久。
- consumer 进程频繁重启。
- CPU 打满。
- GC 抖动。
- 网络抖动。

## 第六步：看下游依赖

Consumer 经常卡在 Kafka 之外：

- 数据库慢查询。
- 连接池耗尽。
- Redis 超时。
- HTTP 下游超时。
- 第三方接口限流。

所有外部调用必须设置 timeout。

## 第七步：看 Partition 与 Consumer 数量

如果 topic 有 3 个 partition，你启动 10 个 consumer，也只有 3 个能真正消费。

扩容前先看：

```text
partition 数量
当前 active consumer 数量
每个 partition lag
```

扩容有效的前提：

```text
consumer 数量 < partition 数量
```

## 第八步：处理坏消息

如果某个 partition lag 卡住不动，可能有坏消息。

处理策略：

1. 根据日志找到 topic、partition、offset。
2. 查看该消息内容。
3. 判断是否可修复。
4. 不可修复则写 DLQ 并提交 offset。
5. 可修复则修复数据或代码后重试。

不要直接跳 offset，除非你已经备份消息并确认业务可接受。

## 决策树

```text
lag 高
  |
  |-- 写入流量突增？
  |     |-- 是：评估恢复时间，必要时临时扩容
  |     |-- 否：继续
  |
  |-- 错误率升高？
  |     |-- 是：按错误类型修复，必要时 DLQ
  |     |-- 否：继续
  |
  |-- handler 变慢？
  |     |-- 是：查数据库、HTTP、锁、慢消息
  |     |-- 否：继续
  |
  |-- 频繁 rebalance？
  |     |-- 是：查心跳、poll 间隔、进程重启
  |     |-- 否：继续
  |
  |-- partition 不够？
        |-- 是：评估扩 partition 的顺序性影响
        |-- 否：查热点 key 或 broker 资源
```

## 临时止血手段

- 暂停非关键 producer。
- 临时增加 consumer，但不能超过 partition 上限。
- 临时提高数据库连接池，但要防止打垮数据库。
- 将坏消息转入 DLQ。
- 对慢下游降级。

## 根因修复

- 优化 handler。
- 批量写入。
- 拆分热点 key。
- 增加 partition。
- 改造 retry/DLQ。
- 给所有外部调用加 timeout 和 circuit breaker。
- 完善监控和告警。

## 本节验收

你需要能完成：

1. 使用命令查看某个 group 的 lag。
2. 判断 lag 是全局高还是单 partition 高。
3. 写出 lag 高的五类原因。
4. 说明为什么不能只靠增加 consumer 解决 lag。
5. 写一份订单消费者 lag 升高排查记录。

