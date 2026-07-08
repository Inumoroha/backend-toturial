# 01 学习 Kafka 前需要补齐什么

Kafka 是一个工程味很重的中间件。它不难入门，但要真正用于 Go 后端项目，你需要同时理解应用代码、网络、磁盘、并发和分布式系统的一些基本概念。

## Go 后端基础

你不需要先成为 Go 专家，但下面这些能力必须比较熟：

- `context.Context`：用于超时、取消、请求生命周期控制。
- goroutine：理解并发执行和资源泄漏风险。
- channel：知道如何做信号通知和任务分发。
- error handling：会区分可重试错误和不可重试错误。
- structured logging：日志中要能带上 `topic`、`partition`、`offset`、`key`、`event_id`。
- graceful shutdown：服务退出前要停止拉取新消息，处理完当前消息，再提交 offset。

### 为什么这些对 Kafka 很重要

Kafka consumer 常常是一个长期运行的后台进程。它不像普通 HTTP handler 那样处理完一个请求就结束。如果你没有做好 context、信号处理、goroutine 生命周期管理，就容易出现：

- 服务已经收到退出信号，但消息还没处理完。
- offset 已经提交，业务却没落库。
- consumer 退出时没有 close，导致 rebalance 变慢。
- 重试逻辑没有上限，某条坏消息把整个消费者卡住。

## Docker 与本地环境

你需要掌握：

- `docker compose up -d`
- `docker compose down`
- 查看容器日志
- 进入容器执行命令
- 理解容器名、端口映射、volume

Kafka 依赖 broker、controller、网络地址和持久化目录。用 Docker Compose 可以让你稳定复现实验环境。

## Linux 与网络基础

建议了解：

- TCP 连接是什么。
- 端口监听是什么。
- 客户端如何连接 broker。
- 磁盘顺序写为什么快。
- 文件描述符过少会造成什么问题。

常用命令：

```powershell
docker ps
docker logs kafka
docker exec -it kafka bash
```

如果你在 Linux 或 macOS：

```bash
ss -lntp
lsof -i :9092
df -h
top
```

## 分布式系统基础

学习 Kafka 时最重要的分布式概念：

- replica：副本。
- leader：负责读写的主副本。
- follower：跟随 leader 复制数据的副本。
- quorum：多数派思想，了解即可。
- failover：leader 挂了之后发生切换。
- at-most-once：最多一次，可能丢消息。
- at-least-once：至少一次，可能重复。
- exactly-once：精确一次，但有严格边界。

## 你暂时不需要深入的内容

这些可以后面再学：

- Kafka 源码。
- Kafka Streams 详细 API。
- Kafka Connect 插件开发。
- 大规模集群运维。
- 跨机房复制。

先把本地实验、Go 客户端、可靠消费做好，这些对 Go 后端求职和工作更直接。

## 本节练习

1. 写一段 Go 程序，监听 `SIGINT` 和 `SIGTERM`，收到信号后优雅退出。
2. 写一个带 `context.WithTimeout` 的函数，模拟超过 2 秒自动取消。
3. 用自己的话解释：为什么 Kafka consumer 不能随便在业务处理前提交 offset？

