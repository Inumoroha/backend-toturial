# 第 7 阶段：架构取舍与高级能力

> 本阶段目标：学会判断 RabbitMQ 适不适合某个业务场景，并能在 RabbitMQ、Kafka、Redis Streams、数据库任务表之间做清晰取舍。同时理解 Classic Queue、Quorum Queue、Stream、RPC、事件驱动、任务队列等高级边界。

## 学习顺序

请按下面顺序学习：

1. [01-本阶段目标与架构取舍思维.md](./01-本阶段目标与架构取舍思维.md)
2. [02-RabbitMQ适用场景与边界.md](./02-RabbitMQ适用场景与边界.md)
3. [03-RabbitMQ-vs-Kafka.md](./03-RabbitMQ-vs-Kafka.md)
4. [04-RabbitMQ-vs-Redis-Streams.md](./04-RabbitMQ-vs-Redis-Streams.md)
5. [05-RabbitMQ-vs-数据库任务表.md](./05-RabbitMQ-vs-数据库任务表.md)
6. [06-Classic-Quorum-Stream队列类型取舍.md](./06-Classic-Quorum-Stream队列类型取舍.md)
7. [07-任务队列-事件驱动-RPC的边界.md](./07-任务队列-事件驱动-RPC的边界.md)
8. [08-吞吐-延迟-顺序-回放-大消息取舍.md](./08-吞吐-延迟-顺序-回放-大消息取舍.md)
9. [09-架构选型决策表.md](./09-架构选型决策表.md)
10. [10-典型业务场景选型案例.md](./10-典型业务场景选型案例.md)
11. [11-演进路线与迁移策略.md](./11-演进路线与迁移策略.md)
12. [12-面试与项目表达.md](./12-面试与项目表达.md)
13. [13-第7阶段练习与自测.md](./13-第7阶段练习与自测.md)

## 建议学习时间

建议用 8 到 12 天完成：

- 第 1 天：建立架构取舍思维。
- 第 2 天：总结 RabbitMQ 适用场景和边界。
- 第 3 天：对比 RabbitMQ 和 Kafka。
- 第 4 天：对比 RabbitMQ 和 Redis Streams。
- 第 5 天：对比 RabbitMQ 和数据库任务表。
- 第 6 天：理解 Classic、Quorum、Stream 队列类型。
- 第 7 天：梳理任务队列、事件驱动和 RPC 的边界。
- 第 8 天：学习吞吐、延迟、顺序、回放、大消息等维度。
- 第 9 到 10 天：完成选型案例、迁移策略和自测。

## 本阶段你要掌握什么

完成本阶段后，你应该能做到：

- 判断 RabbitMQ 适合和不适合哪些场景。
- 解释 RabbitMQ 和 Kafka 的核心差异。
- 解释 RabbitMQ 和 Redis Streams 的核心差异。
- 判断什么时候数据库任务表更简单。
- 在 Classic Queue、Quorum Queue、Stream 之间做基本选择。
- 判断一个场景应该用任务队列、事件驱动、RPC 还是同步调用。
- 从吞吐、延迟、顺序、回放、可靠性、运维复杂度等维度做架构取舍。
- 在面试或项目汇报中清楚表达 RabbitMQ 选型理由。

## 选型心法

不要问：

```text
哪个中间件最好？
```

要问：

```text
我的业务更需要复杂路由、任务分发、低延迟、事件回放、高吞吐、强事务一致性，还是低运维复杂度？
```

架构取舍不是给技术排名，而是让业务风险和技术能力匹配。

## 官方资料

- [RabbitMQ Queues](https://www.rabbitmq.com/docs/queues)
- [RabbitMQ Classic Queues](https://www.rabbitmq.com/docs/classic-queues)
- [RabbitMQ Quorum Queues](https://www.rabbitmq.com/docs/quorum-queues)
- [RabbitMQ Streams](https://www.rabbitmq.com/docs/streams)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Redis Streams](https://redis.io/docs/latest/develop/data-types/streams/)
- [PostgreSQL SELECT](https://www.postgresql.org/docs/current/sql-select.html)
- [MySQL Locking Reads](https://dev.mysql.com/doc/en/innodb-locking-reads.html)

