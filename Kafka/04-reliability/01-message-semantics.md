# 01 消息语义：最多一次、至少一次、精确一次

本节目标：理解 Kafka 中常说的消息语义，并把它们翻译成 Go 后端业务代码中的风险。

## At Most Once

最多一次：消息最多被处理一次，可能丢。

典型情况：

```text
consumer 拉到消息
先提交 offset
业务处理失败或进程崩溃
消息不会再被处理
```

适合：

- 可以接受丢失的日志。
- 非关键埋点。

不适合：

- 订单。
- 支付。
- 库存。
- 优惠券。

## At Least Once

至少一次：消息不会轻易丢，但可能重复。

典型情况：

```text
consumer 处理业务成功
提交 offset 前崩溃
重启后重新消费同一条消息
业务被执行第二次
```

这是很多 Go 后端 Kafka 项目的默认目标。

关键要求：consumer 业务必须幂等。

## Exactly Once

精确一次经常被误解。

Kafka 的 exactly once 主要解决 Kafka 内部和 Kafka 到 Kafka 流转中的部分问题，例如：

- producer 幂等写入。
- Kafka 事务。
- consume-transform-produce 这种链路中的原子提交。

但它不能自动保证你的外部业务只执行一次。

例如 consumer 调用第三方支付接口，Kafka 无法替你保证这个支付接口不会被重复调用。

## Go 后端里的真实目标

真实项目通常追求：

```text
Kafka 层至少一次 + 业务层幂等 = 用户视角接近精确一次
```

换句话说：

- Kafka 可以重复投递。
- 你的业务代码要能识别重复。
- 外部副作用要有幂等键。

## 消息丢失的常见位置

### Producer 侧

- 发送失败但业务没有感知。
- `acks=0`。
- 没有 flush 就退出。
- 发送超时后没有补偿。

### Broker 侧

- 副本不足。
- leader 崩溃且数据未复制。
- topic retention 太短，consumer 还没消费消息已过期。

### Consumer 侧

- 业务未成功就提交 offset。
- 自动提交太早。
- 错误处理直接吞掉。

## 重复消费的常见位置

- producer 重试。
- consumer 处理成功但提交 offset 失败。
- rebalance。
- consumer 崩溃重启。
- retry topic 重放。
- 人工修复后重放 DLQ。

## 本节练习

1. 分别写出最多一次、至少一次的消费伪代码。
2. 解释为什么至少一次一定要求业务幂等。
3. 思考：发优惠券事件重复消费，会造成什么后果？

