# Go 后端工程师 Kafka 教程

这是一套从零开始、面向 Go 后端工程师的 Kafka 系统教程。

目前推荐优先阅读新版中文阶段目录，它会按 `PostgreSQL` 目录中的教程风格持续完善：中文阶段名、编号文件、每篇细讲目标、直觉、命令、代码、常见错误、Go 后端视角、练习和小结。

新版路线入口：[Kafka学习路线_新版.md](Kafka学习路线_新版.md)

面试复习入口：[Kafka面试题与参考答案.md](Kafka面试题与参考答案.md)

模拟面试入口：[Kafka模拟面试.md](Kafka模拟面试.md)

考前速查入口：[Kafka速查表.md](Kafka速查表.md)

## 如何学习

新版建议按中文阶段目录推进：

1. [第1阶段：环境搭建与Kafka直觉](第1阶段：环境搭建与Kafka直觉)
2. [第2阶段：Kafka核心概念与存储模型](第2阶段：Kafka核心概念与存储模型)
3. [第3阶段：Producer生产者机制](第3阶段：Producer生产者机制)
4. [第4阶段：Consumer消费者机制](第4阶段：Consumer消费者机制)
5. [第5阶段：Go接入Kafka](第5阶段：Go接入Kafka)
6. [第6阶段：可靠性、幂等与一致性](第6阶段：可靠性、幂等与一致性)
7. [第7阶段：性能调优与可观测性](第7阶段：性能调优与可观测性)
8. [第8阶段：生产实践与架构设计](第8阶段：生产实践与架构设计)
9. [第9阶段：项目落地：电商事件驱动后端](第9阶段：项目落地：电商事件驱动后端)

旧版英文编号目录仍然保留，作为早期草稿和补充资料。

学习时建议：

1. 先读本阶段 `0_...总览.md`，知道目标、产出和学习节奏。
2. 再按 `1_...md`、`2_...md` 的顺序学习。
3. 每读完一篇，都完成文末练习。
4. 不要跳过实验。Kafka 的很多问题只有亲手制造过故障，才会真正理解。

学习过程中可以用 [learning-checklist.md](learning-checklist.md) 跟踪阶段产出。

## 旧版目录

| 阶段 | 文件夹 | 主题 |
| --- | --- | --- |
| 第 0 阶段 | `00-preparation` | 环境准备、Go 前置知识、Docker Compose |
| 第 1 阶段 | `01-kafka-core` | Kafka 核心概念、Topic、Partition、Offset、Consumer Group |
| 第 2 阶段 | `02-producer-consumer` | Producer、Consumer、提交 offset、重平衡 |
| 第 3 阶段 | `03-go-client` | Go Kafka 客户端、生产者、消费者组、工程封装 |
| 第 4 阶段 | `04-reliability` | 消息可靠性、幂等、事务、重试、死信队列 |
| 第 5 阶段 | `05-performance-observability` | 性能调优、压测、监控、Lag 排查 |
| 第 6 阶段 | `06-production-architecture` | 生产运维、容量规划、安全、事件驱动架构 |
| 第 7 阶段 | `07-project-practice` | 电商事件驱动项目实战 |

## 本教程的学习目标

学完之后，你应该能做到：

- 独立搭建本地 Kafka 学习环境。
- 能解释 Kafka 的 broker、topic、partition、replica、offset、consumer group。
- 能用 Go 写可靠的 Kafka producer 和 consumer。
- 能处理重复消费、消息丢失、消费失败、消息顺序、业务幂等。
- 能设计 retry topic、dead letter topic、outbox pattern。
- 能监控 consumer lag、吞吐、延迟、错误率。
- 能针对订单、库存、支付、通知等后端业务设计事件驱动链路。

## 教程质量说明

如果你想了解这套教程目前覆盖了什么、后来又补强了哪些地方，可以看 [quality-review.md](quality-review.md)。这份审计记录会帮助你判断哪些章节是概念讲解，哪些章节是动手实验或项目施工图。

## 推荐节奏

- 基础薄弱：12 周。
- 有 Go 后端经验但没用过 Kafka：8 周。
- 已在项目中接触过消息队列：4 到 6 周复习加实战。

不要把目标定成“快速看完”。更好的目标是：每个阶段至少写一段代码、跑一个实验、记录一次你亲眼看到的现象。

## 外部资料

- Apache Kafka 官方文档：<https://kafka.apache.org/documentation/>
- Apache Kafka Quickstart：<https://kafka.apache.org/quickstart>
- Apache Kafka Operations：<https://kafka.apache.org/documentation/#operations>
- Confluent Go Client：<https://docs.confluent.io/kafka-clients/go/current/overview.html>
- franz-go：<https://github.com/twmb/franz-go>
- Sarama：<https://github.com/IBM/sarama>
