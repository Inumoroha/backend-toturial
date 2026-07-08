# 01 Go Kafka 客户端如何选择

Go 生态中常见 Kafka 客户端有三个：`confluent-kafka-go`、`franz-go`、`sarama`。学习时不需要一开始全都精通，但要知道它们的差异。

## 常见客户端对比

| 客户端 | 类型 | 优点 | 注意点 |
| --- | --- | --- | --- |
| `confluent-kafka-go` | 基于 librdkafka | 成熟、性能强、功能完整 | 依赖 C 库，构建环境稍复杂 |
| `franz-go` | 纯 Go | API 现代、功能完整、无 C 依赖 | 团队需要接受相对新的使用风格 |
| `sarama` | 纯 Go | 老牌、存量项目多 | API 历史包袱较多，维护策略要关注 |

## 新手建议

如果你是为了学习 Go 后端 Kafka：

1. 选择 `franz-go` 或 `confluent-kafka-go` 做主线。
2. 学会一个后，再读懂 `sarama` 项目代码。
3. 面试时重点讲清楚 Kafka 机制，不要只讲某个库的 API。

## 选择标准

### 公司已有技术栈

如果公司已经统一使用某个客户端，优先跟随公司。

工程中“统一”常常比“个人偏好”更重要。

### 是否接受 CGO

`confluent-kafka-go` 依赖 librdkafka，能力强，但构建和部署要考虑 CGO。

如果你的部署环境希望纯静态、镜像更简单，纯 Go 客户端会更省心。

### 功能需求

确认客户端是否支持：

- consumer group。
- 手动提交 offset。
- 幂等 producer。
- 事务。
- SASL/TLS。
- metrics。
- admin API。

## 本教程推荐实践路线

主线建议使用 `franz-go` 或 `confluent-kafka-go`。

但教程中的工程原则不绑定具体客户端：

- producer 要统一封装。
- consumer 要手动提交。
- 消息要带 event id。
- 错误要分类。
- 退出要优雅。
- 监控要接入。

这些原则比具体 API 更长久。

## 本节练习

1. 阅读三个客户端的 README。
2. 记录它们如何创建 producer。
3. 记录它们如何加入 consumer group。
4. 选择一个作为你的主线客户端，并写下选择理由。

