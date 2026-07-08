# 03 Rebalance 与 Consumer Lag

本节目标：理解 consumer group 为什么会重平衡，以及 lag 升高时该怎样分析。

## Rebalance 是什么

rebalance 是 consumer group 内 partition 分配发生变化的过程。

常见触发原因：

- 新 consumer 加入 group。
- consumer 离开 group。
- consumer 心跳超时。
- topic partition 数量变化。
- consumer 处理太慢，超过最大 poll 间隔。

## Rebalance 的影响

rebalance 期间，consumer group 的消费可能短暂停顿。

如果频繁 rebalance，会导致：

- 消费吞吐下降。
- lag 持续升高。
- 重复消费概率增加。
- 日志中出现大量 group coordination 信息。

## Go consumer 为什么容易触发 rebalance

典型原因：

- 单条消息处理时间太长。
- 业务 handler 调外部接口没有超时。
- goroutine 卡住。
- GC 或 CPU 抖动导致心跳异常。
- 没有优雅退出，进程被强杀。

## 关键参数

不同客户端名称可能略有差异，但核心思想一致。

### session.timeout.ms

consumer 多久没心跳会被认为死亡。

### heartbeat.interval.ms

发送心跳的间隔。

### max.poll.interval.ms

两次 poll 之间允许的最大时间。

如果业务处理太久，超过这个间隔，broker 会认为 consumer 不健康，引发 rebalance。

## Consumer Lag 是什么

lag 表示 consumer group 还有多少消息没处理。

```text
lag = log end offset - committed offset
```

lag 不是坏事本身。短时间升高后能下降，通常没问题。真正要关注的是：

- lag 持续升高。
- lag 长时间不下降。
- 某几个 partition lag 特别高。
- lag 升高伴随错误率升高。

## Lag 排查顺序

### 第一步：看 producer 是否突然增加

如果写入流量暴涨，consumer 处理能力没变，lag 会升高。

### 第二步：看 consumer 是否报错

大量错误重试会拖慢消费。

### 第三步：看下游是否变慢

consumer 常常卡在：

- 数据库。
- Redis。
- HTTP RPC。
- 文件系统。
- 第三方服务。

### 第四步：看 partition 分布

如果只有一个 partition lag 很高，可能是：

- key 热点。
- 某个 partition 上有坏消息。
- 某个 consumer 实例异常。

### 第五步：看 rebalance

频繁 rebalance 会让消费一直不稳定。

## 如何降低 lag

常见手段：

- 优化业务处理耗时。
- 批量写数据库。
- 增加 consumer 实例，但不能超过 partition 并行上限。
- 增加 partition，但要评估顺序性影响。
- 把慢任务拆到异步 worker。
- 对坏消息使用死信队列，不要卡住主链路。

## 本节练习

1. 启动一个 consumer group。
2. 发送 1000 条消息。
3. 在 consumer 中模拟每条消息 sleep 1 秒。
4. 观察 lag 如何增长。
5. 增加 consumer 数量，观察 lag 是否下降。
6. 让 consumer 数量超过 partition 数量，观察是否继续提升。

