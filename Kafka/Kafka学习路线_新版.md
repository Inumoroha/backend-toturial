# Kafka 学习路线（PostgreSQL 教程风格新版）

这份路线用于替代早期较短的 `kafka-go-backend-roadmap.md`。新版教程会按 PostgreSQL 目录的写法展开：中文阶段目录、编号文件、每篇包含目标、直觉、命令、代码、解释、常见错误、Go 后端视角、练习和小结。

---

## 一、阶段总览

| 阶段 | 目录 | 目标 |
| --- | --- | --- |
| 第 1 阶段 | `第1阶段：环境搭建与Kafka直觉` | 启动 Kafka，完成第一次消息收发，建立 topic、partition、offset 直觉 |
| 第 2 阶段 | `第2阶段：Kafka核心概念与存储模型` | 理解 broker、controller、replica、ISR、retention、log compaction |
| 第 3 阶段 | `第3阶段：Producer生产者机制` | 掌握 key、partitioner、acks、batch、linger、压缩、幂等生产者 |
| 第 4 阶段 | `第4阶段：Consumer消费者机制` | 掌握 consumer group、offset 提交、rebalance、lag、失败处理 |
| 第 5 阶段 | `第5阶段：Go接入Kafka` | 使用 Go 编写 producer、consumer、封装基础包、接入 context 和日志 |
| 第 6 阶段 | `第6阶段：可靠性、幂等与一致性` | 处理消息丢失、重复消费、业务幂等、retry、DLQ、outbox |
| 第 7 阶段 | `第7阶段：性能调优与可观测性` | 做压测、看 lag、调 producer/consumer 参数、接 Prometheus 指标 |
| 第 8 阶段 | `第8阶段：生产实践与架构设计` | topic 规划、容量估算、安全、ACL、schema 演进、上线评审 |
| 第 9 阶段 | `第9阶段：项目落地：电商事件驱动后端` | 完成 Go + Kafka 电商事件驱动项目 |

---

## 二、建议学习节奏

如果你是 Go 后端初学者，建议用 10 到 12 周：

```text
第 1 周：第 1 阶段
第 2 周：第 2 阶段
第 3 周：第 3 阶段
第 4 周：第 4 阶段
第 5-6 周：第 5 阶段
第 7 周：第 6 阶段
第 8 周：第 7 阶段
第 9 周：第 8 阶段
第 10-12 周：第 9 阶段项目
```

如果你已经熟悉 RabbitMQ、Redis Stream 或其他消息队列，可以压缩到 6 到 8 周。

---

## 三、每阶段最终产出

| 阶段 | 产出 |
| --- | --- |
| 第 1 阶段 | 能用命令行完成消息收发并解释 lag |
| 第 2 阶段 | 能画出 Kafka topic、partition、replica、offset 模型 |
| 第 3 阶段 | 能解释 producer 参数如何影响可靠性和吞吐 |
| 第 4 阶段 | 能设计手动提交 offset 的 consumer 流程 |
| 第 5 阶段 | 能写 Go producer 和 consumer group |
| 第 6 阶段 | 能实现幂等消费、retry topic、DLQ、outbox |
| 第 7 阶段 | 能完成压测并写 lag 排查记录 |
| 第 8 阶段 | 能写 Kafka 链路上线评审 |
| 第 9 阶段 | 能完成一个可展示的电商事件驱动后端项目 |

---

## 四、学习原则

Kafka 的学习原则是：

```text
每个概念都要跑实验。
每个实验都要看现象。
每个现象都要能映射到 Go 后端代码。
每个失败场景都要有恢复策略。
```

不要满足于“知道 Kafka 是消息队列”。一个 Go 后端工程师真正需要的是：

- 知道什么时候会丢消息。
- 知道什么时候会重复消费。
- 知道什么时候提交 offset。
- 知道如何做业务幂等。
- 知道 lag 高了怎么查。
- 知道如何把 Kafka 放进真实项目。

