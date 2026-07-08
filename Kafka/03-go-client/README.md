# 第 3 阶段：Go 客户端开发

这一阶段开始把 Kafka 放进 Go 后端项目里。你会学习客户端选择、producer 封装、consumer group 编写、优雅退出和工程目录设计。

## 本阶段目标

完成后你应该能做到：

- 选择适合项目的 Go Kafka 客户端。
- 封装一个可复用 producer。
- 封装一个支持手动提交的 consumer group。
- 处理 context、日志、错误、重试、关闭。
- 为后续项目实战准备基础代码结构。

## 学习顺序

1. [01-client-selection.md](01-client-selection.md)
2. [02-producer-wrapper.md](02-producer-wrapper.md)
3. [03-consumer-group-wrapper.md](03-consumer-group-wrapper.md)
4. [04-project-structure.md](04-project-structure.md)
5. [05-franz-go-hands-on.md](05-franz-go-hands-on.md)

## 建议耗时

2 周。

建议你优先选择一个客户端深入实践，不要同时纠结三个库的 API。
