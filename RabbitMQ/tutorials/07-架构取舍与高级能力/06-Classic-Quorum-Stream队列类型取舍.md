# 06. Classic、Quorum、Stream 队列类型取舍

## 1. 为什么要区分队列类型

RabbitMQ 不只有一种队列。

常见类型：

- Classic Queue。
- Quorum Queue。
- Stream。

不同队列适合不同场景。

不要所有场景都默认一种队列。

## 2. Classic Queue

Classic Queue 是传统队列。

适合：

- 学习。
- 普通任务队列。
- 对复制要求不高的业务。
- 延迟敏感、流量中等的工作负载。

常见使用：

```text
notification.email.worker.queue
report.generate.queue
image.compress.queue
```

## 3. Classic Queue 的注意点

如果是单副本 classic queue：

```text
队列所在节点故障会影响该队列。
```

生产环境要根据业务重要性判断是否需要 quorum queue。

Classic Queue 不是不能用于生产，而是要清楚它的可靠性边界。

## 4. Quorum Queue

Quorum Queue 面向高可用和数据安全。

它基于复制日志思想，适合关键业务消息。

适合：

- 支付事件。
- 订单关键状态事件。
- 资金流水消息。
- 不能接受单节点队列丢失风险的业务。

## 5. Quorum Queue 的代价

Quorum Queue 不是免费增强。

代价：

- 写入成本更高。
- 磁盘和网络开销更大。
- 延迟可能增加。
- 对集群网络更敏感。
- 不适合大量临时队列。

所以不要所有队列都无脑 quorum。

## 6. Stream

RabbitMQ Stream 是日志式流模型。

适合：

- 高吞吐事件流。
- 事件保留。
- 重复读取。
- 多消费者从不同位置读取。

它更接近事件流，不是普通任务队列的直接替代。

如果你需要：

```text
消息 ack 后仍然保留
消费者可以回放历史事件
```

可以评估 Stream。

## 7. 三者对比

| 类型 | 更适合 | 不适合 |
| --- | --- | --- |
| Classic Queue | 普通任务队列、基础业务消息 | 高数据安全复制要求 |
| Quorum Queue | 关键业务消息、高可用队列 | 大量临时队列、极致低开销 |
| Stream | 高吞吐事件流、保留和回放 | 普通一次性任务处理 |

## 8. 怎么选择

### 普通通知任务

```text
Classic Queue
```

原因：

```text
失败可重试，偶发延迟可接受，不一定需要复制队列。
```

### 支付成功事件

```text
Quorum Queue
```

原因：

```text
业务关键，消息安全更重要。
```

### 用户行为日志流

```text
Kafka 或 RabbitMQ Stream
```

原因：

```text
需要保留和回放。
```

## 9. 和 Kafka 的 Stream 场景区别

RabbitMQ Stream 可以承载流式场景。

但如果公司已有成熟 Kafka 平台，并且目标是实时数仓、日志分析、海量事件流，Kafka 仍然常见。

选择时要看：

- 现有平台。
- 团队能力。
- 吞吐规模。
- 回放需求。
- 和现有 RabbitMQ 拓扑的整合。

## 10. 本节小结

队列类型选择：

```text
普通业务任务 -> Classic
关键可靠业务 -> Quorum
事件流保留回放 -> Stream 或 Kafka
```

核心是根据业务风险和成本选择，不是追最新特性。

下一节学习任务队列、事件驱动和 RPC 的边界。

