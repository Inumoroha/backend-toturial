# 第 5 阶段：Go 接入 Kafka 总览

这一阶段把 Kafka 放进 Go 后端项目中。你会学习如何选择客户端、如何封装 producer、如何封装 consumer group、如何处理 context、日志、错误、配置和优雅退出。

---

## 一、本阶段课程

1. `1_Go_Kafka客户端选择.md`
2. `2_项目结构与配置设计.md`
3. `3_封装Producer.md`
4. `4_封装ConsumerGroup.md`
5. `5_错误分类Retry与DLQ代码边界.md`
6. `6_综合实践_Go完成订单事件收发.md`

---

## 二、本阶段学习目标

完成后你应该能做到：

- 选择一个适合项目的 Go Kafka 客户端。
- 写出 producer wrapper。
- 写出 consumer handler 接口。
- 使用 context 控制发送和消费生命周期。
- 使用结构化日志记录 topic、partition、offset、event_id。
- 处理 SIGTERM，优雅退出 consumer。

---

## 三、客户端选择建议

常见选择：

- `franz-go`：纯 Go，适合学习和现代 Go 项目。
- `confluent-kafka-go`：基于 librdkafka，成熟，性能强。
- `sarama`：老牌纯 Go 客户端，存量项目多。

本阶段主线建议使用 `franz-go`，因为它没有 CGO 依赖，学习时环境更简单。

---

## 四、本阶段最终产出

学完本阶段，你不应该只会写一段临时 demo，而应该能得到一套可以放进 Go 项目的 Kafka 接入骨架：

```text
internal/kafka/
  message.go
  producer.go
  consumer.go
  retry.go
  dlq.go
internal/order/
  event.go
  handler.go
cmd/order-producer/
  main.go
cmd/inventory-consumer/
  main.go
```

这套骨架要做到：

- 业务层不直接依赖具体 Kafka 客户端。
- Producer 支持 `context.Context`、key、headers、错误返回。
- ConsumerGroup 支持 handler、手动提交 offset、优雅退出。
- 错误能区分可重试和不可重试。
- 日志能打印 `topic`、`partition`、`offset`、`key`、`event_id`。

也就是说，本阶段的目标是“工程封装”，不是“能发一条消息就结束”。

---

## 五、最小闭环

本阶段最小闭环是：

```text
启动 Kafka
-> Go producer 发布 order.created
-> Go consumer group 消费 order.created
-> handler 打印 event_id
-> handler 成功后提交 offset
-> 再次启动 consumer 不重复消费已提交消息
```

验证命令：

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group inventory-service
```

你应该能看到 group 的 offset 向前移动，lag 逐步变成 `0`。

---

## 六、学习顺序建议

建议按下面节奏学习：

1. 先比较 Go Kafka 客户端，知道为什么本教程主线选 `franz-go`。
2. 再搭项目结构，把配置、日志、数据库和 Kafka 初始化位置固定下来。
3. 然后封装 Producer，重点关注 key、headers、发送错误。
4. 再封装 ConsumerGroup，重点关注 handler 边界和 offset 提交。
5. 最后把 Retry/DLQ 的代码边界设计好，避免业务 handler 到处复制失败处理逻辑。

不要一开始就写一个“大而全”的 Kafka 包。先让接口小而清楚，再逐步补能力。

---

## 七、Go 后端常见错误

### 1. 在业务代码里到处 new Kafka client

这样会导致连接资源混乱，也不利于测试。Kafka client 应该在 `main.go` 组装，再注入业务 service。

### 2. Producer 封装吞掉错误

如果 `Publish` 内部只打印日志然后返回 `nil`，业务层会以为消息发送成功，最终造成消息丢失。

### 3. Consumer handler 自己提交 offset

handler 应该只处理业务并返回错误。offset 提交由 consumer 框架统一处理，这样 retry、DLQ、日志和指标才有一致规则。

### 4. 忽略退出信号

Go 服务在容器环境里经常收到 `SIGTERM`。consumer 如果不能优雅退出，容易在 rebalance 或 offset 提交阶段制造重复消费。

---

## 八、本阶段验收

完成本阶段后，用下面问题自测：

- 我能否解释为什么业务层应该依赖 Kafka 接口，而不是直接依赖第三方 client？
- 我能否写出一个 `Producer.Publish(ctx, topic, msg)` 接口？
- 我能否说明 key 和 headers 应该放什么？
- 我能否写出 consumer handler 的接口边界？
- 我能否说明 handler 成功、可重试失败、不可重试失败时分别如何提交 offset？
- 我能否通过命令观察 consumer group offset 和 lag？

这些问题能回答清楚，再进入第 6 阶段可靠性设计会顺很多。

---

## 九、本阶段推荐代码提交顺序

建议按下面顺序写代码：

1. 创建 Go module 和目录结构。
2. 实现配置加载。
3. 定义 `kafka.Message`。
4. 定义 `Producer` 接口。
5. 实现一个真实 producer。
6. 定义 `Handler` 接口。
7. 实现 consumer group runner。
8. 增加错误分类。
9. 增加结构化日志字段。
10. 写综合实践：订单事件收发。

每一步都要能 `go test ./...` 或至少 `go test` 编译通过。

---

## 十、推荐接口草图

本阶段你最终应该能写出类似接口：

```go
type Message struct {
    Key     []byte
    Value   []byte
    Headers map[string]string
}

type Producer interface {
    Publish(ctx context.Context, topic string, msg Message) error
}

type Handler interface {
    Handle(ctx context.Context, msg Message) error
}
```

接口越小，业务层越容易测试。不要把底层客户端的复杂配置直接泄漏给业务代码。

---

## 十一、本阶段测试策略

至少准备三类测试：

| 测试 | 目的 |
| --- | --- |
| 单元测试 | 验证事件构造、错误分类、配置加载 |
| fake producer 测试 | 验证业务层调用 producer 的参数 |
| 集成测试 | 验证消息真的进入 Kafka 并被 consumer 读到 |

学习阶段可以先写 fake producer，不必一开始就搭完整集成测试。

---

## 十二、阶段完成标准

你可以进入第 6 阶段的标志是：

- 能写出统一 producer wrapper。
- 能写出统一 consumer handler 接口。
- 能解释业务层为什么不直接依赖 Kafka 客户端。
- 能在日志里输出 topic、partition、offset、key、event_id。
- 能处理 context 取消和服务退出。
- 能让一条 `order.created` 从 Go producer 到 Go consumer 完整流转。

---

## 十三、推荐复盘问题

完成本阶段后，写下：

```text
我的 producer wrapper 对业务层暴露了哪些方法？
业务代码是否还能直接接触 Kafka client？
consumer handler 返回错误后，由谁决定是否提交 offset？
context 取消时，producer 和 consumer 如何退出？
日志里是否包含排查 Kafka 消息所需字段？
```

如果这些问题答不清楚，说明代码虽然能跑，但工程边界还不稳。

---

## 十四、下一阶段预告

第 6 阶段会继续追问：

- 消息发送失败怎么办？
- 消息重复消费怎么办？
- 数据库成功但 Kafka 失败怎么办？
- handler 失败后如何 retry？
- 什么消息应该进入 DLQ？

也就是说，第 5 阶段是代码骨架，第 6 阶段是可靠性规则。

---

## 十五、本阶段交付物

建议最终留下：

```text
internal/kafka/message.go
internal/kafka/producer.go
internal/kafka/consumer.go
internal/kafka/error.go
cmd/order-producer/main.go
cmd/inventory-consumer/main.go
```

这些文件后面可以直接迁移到第 9 阶段项目中。

---

## 十六、阶段结束小作业

完成一个最小 Go 程序：

```text
cmd/order-producer 发布 1 条 order.created。
cmd/inventory-consumer 消费并打印 event_id。
consumer 成功后提交 offset。
日志包含 topic、partition、offset、key。
```

这不是最终项目，但它是第 9 阶段项目的 Kafka 接入雏形。

---

## 十七、学习产物检查

本阶段结束时，你的代码或笔记里应该能找到：

- producer 接口定义。
- consumer handler 接口定义。
- message struct。
- 错误分类类型。
- 配置加载函数。
- 一个 producer main。
- 一个 consumer main。
- 一段 consumer group lag 验证记录。

如果只有一段散落在 `main.go` 里的 demo，建议回到 `2_项目结构与配置设计.md` 重新整理。

---

## 十八、面试表达

可以这样描述 Go 接入 Kafka 的工程边界：

```text
我会把 Kafka 客户端封装在 internal/kafka，业务层只依赖 Producer 和 Handler 这类小接口。
Producer 负责统一处理 key、headers、context 和发送错误；Consumer 框架负责拉消息、调用 handler、处理 retry/DLQ 和提交 offset。
业务 handler 不直接提交 offset，也不直接操作底层 Kafka client。
```

---

## 十九、推荐学习时间安排

如果每天学习 1 到 2 小时，可以按 5 天推进：

| 天数 | 任务 | 产出 |
| --- | --- | --- |
| 第 1 天 | 客户端选择、项目结构 | `go.mod`、目录结构 |
| 第 2 天 | Producer 封装 | `producer.go`、发送示例 |
| 第 3 天 | ConsumerGroup 封装 | `consumer.go`、handler 接口 |
| 第 4 天 | 错误分类、Retry/DLQ 边界 | `error.go`、伪代码 |
| 第 5 天 | 综合实践 | Go 订单事件收发闭环 |

不要一天把代码全部堆完。每一步都要能解释为什么这么分层。

---

## 二十、代码审查清单

本阶段代码写完后，自查：

- [ ] `main.go` 只负责组装依赖。
- [ ] 业务包不直接 import 具体 Kafka client。
- [ ] Producer 发送失败会返回 error。
- [ ] Consumer handler 不提交 offset。
- [ ] context 能传到发送和处理逻辑。
- [ ] 日志字段统一。
- [ ] fake producer 可以用于业务单元测试。

如果这些检查项做不到，第 9 阶段项目会越写越乱。

---

## 二十一、阶段结束产物示例

建议最终留下：

```text
docs/go-kafka-wrapper.md
examples/order-producer
examples/inventory-consumer
internal/kafka
```

文档里要写清楚接口边界和 offset 提交策略，方便后续项目复用。

---

## 二十二、最终检查

把 producer 和 consumer 分别关掉重启一次，确认：

- producer 退出前会关闭客户端。
- consumer 重启后不会重复消费已提交消息。
- 日志足够定位消息。

---

## 二十三、下一步连接

第 5 阶段完成的是 Go 代码骨架。第 6 阶段会把这套骨架放进失败场景里，继续补上防丢、幂等、Retry、DLQ 和 Outbox。

---

## 二十四、复习建议

本阶段的代码不要追求花哨抽象。好的 Kafka 封装应该让业务代码更清楚：

```text
我要发布什么事件？
我要处理什么消息？
失败时返回什么错误？
```

如果封装让这些问题更难回答，就说明抽象过度了。

---

## 二十五、代码复盘

完成本阶段后，打开 `internal/kafka` 自查：

```text
这个包是否只包含 Kafka 通用概念？
业务事件是否留在业务包？
测试时是否能替换 fake producer？
```

边界清楚，后续可靠性代码才好加。

---

## 二十二、本阶段最终目录要求

学完本阶段后，你的 Go 项目至少应该能分出这些边界：

```text
config：读取 Kafka 地址、topic、group、超时配置。
producer：封装发送接口，不暴露具体客户端细节。
consumer：封装消费循环、提交 offset、退出控制。
event：定义事件结构、版本、序列化和反序列化。
handler：只写业务处理逻辑。
test：使用 fake producer 或 fake handler 验证边界。
```

这和 PostgreSQL 教程里把 db、repository、service 分开是同一种思想：基础设施代码要稳定，业务代码要清爽。

---

## 二十三、学习时的最小项目

本阶段可以先做一个很小的 Go 项目：

```text
一个命令发送 order.created。
一个命令消费 order.created。
事件结构放在 internal/event。
producer 和 consumer 各有接口封装。
配置从环境变量读取。
测试中使用 fake publisher。
```

项目越小，越容易看清 Kafka 接入边界。等边界稳定后，再加入 HTTP API、数据库和 Outbox。
