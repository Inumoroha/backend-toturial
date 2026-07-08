# 第 5 阶段：性能与可观测性

这一阶段学习如何让 Kafka 链路跑得稳、跑得快、看得见。性能优化不是背参数，而是知道瓶颈在哪里，并能用指标证明你的判断。

## 本阶段目标

完成后你应该能做到：

- 区分吞吐优化和低延迟优化。
- 调整 producer batch、linger、compression。
- 调整 consumer 批量、fetch、并发模型。
- 监控 consumer lag、处理耗时、错误率。
- 设计 Kafka 压测实验。
- 能按步骤排查 lag 持续升高。

## 学习顺序

1. [01-producer-performance.md](01-producer-performance.md)
2. [02-consumer-performance.md](02-consumer-performance.md)
3. [03-metrics-and-alerting.md](03-metrics-and-alerting.md)
4. [04-benchmark-method.md](04-benchmark-method.md)
5. [05-lag-troubleshooting-playbook.md](05-lag-troubleshooting-playbook.md)

## 建议耗时

1 周。

这一阶段必须动手压测。没有数据的调参只是猜。
