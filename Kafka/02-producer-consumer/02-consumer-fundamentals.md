# 02 Consumer 基础

Consumer 的职责是从 Kafka 拉取消息并执行业务逻辑。Go 后端里，consumer 往往是最容易出事故的部分，因为它同时涉及并发、重试、offset、数据库和外部接口。

## Consumer 的基本流程

推荐生产流程：

1. 加入 consumer group。
2. 拉取消息。
3. 解析消息。
4. 校验 schema 和业务字段。
5. 执行业务逻辑。
6. 成功后提交 offset。
7. 失败时进入重试或死信流程。

关键原则：业务成功之后再提交 offset。

## 自动提交的问题

自动提交看起来方便，但容易出现这种情况：

```text
consumer 拉到消息
offset 被自动提交
业务处理还没成功
服务崩溃
重启后从已提交 offset 继续
这条消息丢失处理机会
```

所以生产项目一般使用手动提交。

## 手动提交的基本原则

### 单条处理

适合业务处理较重、可靠性要求高的场景。

```text
拉取一条消息
处理成功
提交 offset
```

优点：清晰可靠。

缺点：吞吐较低。

### 批量处理

适合日志、统计、批量写入数据库。

```text
拉取一批消息
批量处理成功
提交这一批最大 offset
```

难点：

- 批次中间某条失败怎么办。
- 批量写数据库是否部分成功。
- 是否需要拆分重试。

## 消费失败分类

### 可重试错误

例如：

- 数据库临时不可用。
- 下游 HTTP 超时。
- Redis 短暂连接失败。

处理方式：

- 本地短暂重试。
- 发送到 retry topic。
- 不要无限阻塞主消费循环。

### 不可重试错误

例如：

- JSON 格式错误。
- 必填字段缺失。
- schema version 不支持。

处理方式：

- 记录错误。
- 发送到 dead letter topic。
- 提交原消息 offset，避免卡住整个 partition。

## Consumer 并发模型

Go consumer 不要盲目开很多 goroutine 处理同一个 partition 的消息。

原因：

- 同一个 partition 内消息有顺序。
- 并发处理后提交 offset 变复杂。
- 后面的消息先成功，前面的消息失败，会造成提交边界混乱。

常见做法：

- 一个 consumer 实例负责多个 partition。
- 每个 partition 内尽量顺序处理。
- 需要并发时，按 key 或 partition 做有序队列。
- 对耗时任务，可以把任务拆到下游 worker，但 offset 提交要谨慎。

## 优雅退出

consumer 收到退出信号时：

1. 停止拉取新消息。
2. 等当前正在处理的消息完成。
3. 成功则提交 offset。
4. close consumer。
5. 退出进程。

如果直接杀进程，可能导致：

- 已处理但未提交，重启后重复消费。
- 正在处理的外部调用中断。
- consumer group rebalance 变慢。

## 日志字段

每次处理消息，日志至少包含：

- topic
- partition
- offset
- key
- event_id
- group
- handler
- attempt
- duration_ms

这样线上排查时，你才能定位“哪条消息在哪个消费者上失败了”。

## 本节练习

1. 用命令行 consumer group 消费消息。
2. 消费一部分后停止，再重启同一个 group。
3. 观察 offset 是否继续推进。
4. 设计一段伪代码：业务成功后提交 offset，失败后不提交。
5. 思考：如果消息 JSON 解析失败，不提交 offset 会发生什么？

