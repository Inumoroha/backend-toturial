# 1. Go Kafka 客户端选择

本节目标：理解 Go 生态中常见 Kafka 客户端的差异，知道学习阶段和项目阶段应该如何选择，并明确后续教程为什么以 `franz-go` 作为主线。

Go 接入 Kafka 的难点不只是“哪个库能发消息”。真正要考虑的是：部署方式、是否依赖 CGO、consumer group 支持、手动提交 offset、事务支持、性能、社区状态、公司存量项目和团队熟悉度。

---

## 一、Go 生态常见 Kafka 客户端

常见选择有三个：

| 客户端 | 类型 | 特点 |
| --- | --- | --- |
| `franz-go` | 纯 Go | API 现代、功能完整、无 CGO |
| `confluent-kafka-go` | 基于 librdkafka | 成熟、性能强、和 Confluent 生态结合好 |
| `sarama` | 纯 Go | 老牌、存量项目多 |

这三个都可以用于真实项目，但适合的场景不同。

---

## 二、为什么学习阶段推荐 franz-go

本教程后续 Go 代码主线推荐 `franz-go`，原因是：

- 纯 Go，不需要处理 CGO 和 C 库依赖。
- 支持 producer。
- 支持 consumer group。
- 支持手动提交 offset。
- 支持 admin 操作。
- API 设计较现代。
- 适合在 Windows、本地 Docker、普通 Go 项目中学习。

安装：

```powershell
go get github.com/twmb/franz-go/pkg/kgo
```

一个最小客户端大致是：

```go
client, err := kgo.NewClient(
    kgo.SeedBrokers("localhost:9092"),
    kgo.ClientID("order-service"),
)
if err != nil {
    return err
}
defer client.Close()
```

这里的 `SeedBrokers` 就是 Kafka bootstrap brokers。

---

## 三、confluent-kafka-go 适合什么场景

`confluent-kafka-go` 基于 `librdkafka`。

优点：

- 成熟稳定。
- 性能强。
- 功能覆盖完整。
- 和 Confluent Platform、Schema Registry 等生态结合紧密。
- 很多公司生产环境使用。

注意点：

- 依赖 CGO。
- 构建镜像时要考虑 C 库和系统依赖。
- 跨平台开发时，新手可能遇到编译环境问题。

适合：

- 公司已有 Confluent 生态。
- 对性能和成熟度要求很高。
- 团队能处理 CGO 构建和部署。

---

## 四、Sarama 适合什么场景

`sarama` 是 Go 生态里很早的 Kafka 客户端。

优点：

- 纯 Go。
- 存量项目多。
- 很多历史教程和项目都用过。

注意点：

- API 历史包袱较多。
- 不同版本维护策略需要关注。
- 新项目未必是最优先选择。

适合：

- 公司老项目已经使用 Sarama。
- 你需要维护存量服务。
- 面试或阅读老代码时需要看懂。

---

## 五、选择客户端时看什么

不要只问：

```text
哪个库更火？
```

应该看：

```text
公司是否已有统一技术栈？
是否允许 CGO？
部署镜像是否需要极简？
是否需要事务？
是否需要 SASL/TLS？
是否需要 admin API？
是否容易做手动提交 offset？
是否容易接 metrics？
团队是否熟悉？
```

对于学习者：

```text
先选一个主线客户端深入，不要三个一起学。
```

你真正要掌握的是 Kafka 机制：

- key。
- partition。
- offset。
- consumer group。
- manual commit。
- rebalance。
- retry。
- DLQ。
- idempotency。

客户端只是实现这些机制的工具。

---

## 六、Go 项目中不要让客户端 API 到处扩散

错误做法：

```text
order service 里直接 new kafka client
inventory service 里直接 new kafka client
notification service 里直接 new kafka client
每个地方都自己处理发送、日志、错误
```

问题：

- 配置分散。
- 错误处理不一致。
- 日志字段不统一。
- 后续切换客户端很痛苦。
- 测试不好 mock。

推荐做法：

```text
internal/kafka/
  config.go
  producer.go
  consumer.go
  message.go
  retry.go
  dlq.go
```

业务模块只依赖你自己定义的接口：

```go
type Producer interface {
    Publish(ctx context.Context, topic string, msg Message) error
}
```

这样后续即使从 `franz-go` 换成 `confluent-kafka-go`，业务代码也不需要大改。

---

## 七、推荐基础接口

### Message

```go
type Message struct {
    Key     []byte
    Value   []byte
    Headers map[string]string
}
```

业务事件通常再包一层：

```go
type Event struct {
    EventID    string          `json:"event_id"`
    EventType  string          `json:"event_type"`
    Version    int             `json:"version"`
    OccurredAt time.Time       `json:"occurred_at"`
    Data       json.RawMessage `json:"data"`
}
```

### Producer

```go
type Producer interface {
    Publish(ctx context.Context, topic string, msg Message) error
    Close(ctx context.Context) error
}
```

### Handler

```go
type Handler interface {
    Handle(ctx context.Context, msg ConsumedMessage) error
}
```

### ConsumedMessage

```go
type ConsumedMessage struct {
    Topic     string
    Partition int32
    Offset    int64
    Key       []byte
    Value     []byte
    Headers   map[string]string
}
```

这些接口的目的是把 Kafka 客户端细节和业务逻辑分开。

---

## 八、配置设计

建议环境变量：

```text
KAFKA_BROKERS=localhost:9092
KAFKA_CLIENT_ID=order-service
KAFKA_GROUP_ID=inventory-service
KAFKA_SECURITY_PROTOCOL=PLAINTEXT
```

Go 配置结构：

```go
type Config struct {
    Brokers  []string
    ClientID string
    GroupID  string
}
```

加载时注意：

- brokers 是数组，不是单个字符串。
- 本地和生产配置要分开。
- 不要把密码写死在代码里。

---

## 九、学习阶段的选择结论

本教程后续建议：

```text
主线使用 franz-go。
了解 confluent-kafka-go 的适用场景。
能读懂 sarama 存量项目的大致结构。
```

你不需要一开始同时写三套代码。

更好的路径：

1. 用 `franz-go` 写通 producer。
2. 用 `franz-go` 写通 consumer group。
3. 实现手动提交 offset。
4. 实现 retry 和 DLQ。
5. 再去对比其他客户端 API。

---

## 十、常见误区

### 1. 只要会调用 Send 就算会 Kafka

不是。

Go 后端中更重要的是：

- 发送失败怎么办？
- 业务成功前能不能提交 offset？
- 重复消费怎么处理？
- consumer 退出时怎么处理正在消费的消息？

### 2. 纯 Go 一定比 CGO 好

不一定。

纯 Go 部署简单，但 `librdkafka` 非常成熟。选择要看团队和生产环境。

### 3. 客户端选择比 Kafka 机制更重要

不是。

Kafka 机制才是核心。客户端 API 可以换，机制理解不能缺。

---

## 十一、本节练习

1. 打开 `franz-go`、`confluent-kafka-go`、`sarama` 的 README。
2. 分别记录它们如何创建 producer。
3. 分别记录它们如何创建 consumer group。
4. 写下你学习阶段选择哪个客户端，以及理由。
5. 设计你自己的 `Producer` 接口。
6. 设计你自己的 `Handler` 接口。
7. 思考：为什么业务代码不应该直接依赖第三方 Kafka 客户端？

---

## 十二、本节小结

- Go Kafka 客户端常见选择有 `franz-go`、`confluent-kafka-go`、`sarama`。
- 学习阶段推荐使用纯 Go 的 `franz-go` 降低环境复杂度。
- `confluent-kafka-go` 成熟且性能强，但需要处理 CGO。
- `sarama` 存量项目多，适合维护老项目时了解。
- 客户端选择要结合公司技术栈、部署方式、功能需求和团队经验。
- 业务代码不应该到处直接调用第三方 Kafka API。
- 应该在 `internal/kafka` 中封装 producer、consumer、message、retry、DLQ。
- 真正重要的是 Kafka 机制，而不是某个库的 API 细节。

---

## 十三、选择决策表

| 场景 | 推荐倾向 | 原因 |
| --- | --- | --- |
| 学习和纯 Go 项目 | `franz-go` | 无 CGO，现代纯 Go |
| 公司已有 librdkafka 体系 | `confluent-kafka-go` | 成熟、性能强 |
| 维护老项目 | `sarama` | 存量项目常见 |
| 极度重视部署简单 | 纯 Go 客户端 | 避免 CGO 和系统库 |

这不是绝对答案。团队已有经验和运行环境往往比个人偏好更重要。

---

## 十四、选型文档模板

```markdown
# Go Kafka 客户端选型

## 候选项

## 部署约束

## Producer 支持

## ConsumerGroup 支持

## 事务/幂等支持

## 团队熟悉度

## 最终选择
```

写出理由，比简单说“这个库更好”更专业。

---

## 二十、选择库时要问的问题

选择 Kafka Go 客户端时，不要只看 GitHub star。

```text
是否支持 consumer group。
是否支持手动提交 offset。
是否方便设置 SASL、TLS 等生产配置。
是否能拿到 topic、partition、offset 等元数据。
是否容易做单元测试和接口封装。
项目维护是否活跃。
团队成员是否能理解它的 API。
```

如果只是学习阶段，可以先选择 API 更容易理解的库；如果进入生产项目，就要把认证、监控、性能和维护成本一起纳入评估。
