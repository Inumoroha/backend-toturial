# 第 0 阶段：准备阶段

这一阶段的目标不是马上学习 Kafka API，而是先把学习 Kafka 所需的地基铺好：Go 并发、Docker、本地 Kafka 环境、基础命令、实验记录方法。

## 本阶段目标

完成后你应该能做到：

- 能用 Docker Compose 启动 Kafka。
- 能使用 Kafka 自带命令行工具创建 topic、发送消息、消费消息。
- 能理解学习 Kafka 时必须关注的三个维度：吞吐、可靠性、可恢复性。
- 能准备好后续 Go 项目的目录和工具链。

## 学习顺序

1. [01-learning-prerequisites.md](01-learning-prerequisites.md)
2. [02-local-kafka-env.md](02-local-kafka-env.md)
3. [03-command-line-first-touch.md](03-command-line-first-touch.md)
4. [04-study-method.md](04-study-method.md)
5. [05-environment-verification.md](05-environment-verification.md)

## 建议耗时

3 到 5 天。

不要急着写 Go 客户端。先让自己能用命令行稳定操作 Kafka，这会让后续排查问题轻松很多。
