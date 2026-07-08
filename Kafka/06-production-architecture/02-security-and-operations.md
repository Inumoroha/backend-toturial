# 02 安全与运维基础

生产 Kafka 集群必须考虑安全和运维。即使你是 Go 后端工程师，也应该知道这些机制如何影响应用配置。

## TLS

TLS 用于加密客户端和 broker 之间的通信。

Go 客户端需要配置：

- CA 证书。
- 客户端证书，可选。
- broker 地址。
- server name 验证。

## SASL

SASL 用于身份认证。

常见机制：

- PLAIN
- SCRAM-SHA-256
- SCRAM-SHA-512
- OAUTHBEARER

Go 服务通常通过环境变量注入用户名和密码。

## ACL

ACL 用于授权。

生产中不应该让所有服务拥有所有 topic 权限。

例如：

- `order-service` 可以写 `order.created`。
- `inventory-service` 可以读 `order.created`，写 `inventory.deducted`。
- `notification-service` 可以读多个事件 topic，但不能写订单 topic。

## 运维常见问题

### Broker 宕机

关注：

- partition leader 是否切换成功。
- 是否出现 offline partitions。
- ISR 是否下降。
- producer 是否大量超时。

### 磁盘打满

可能原因：

- retention 设置过长。
- 消息量增长。
- 副本数过多。
- consumer lag 长期不处理不是直接原因，但历史保留策略可能使数据无法及时清理。

### Consumer Lag 飙升

排查：

- producer 流量。
- consumer 错误。
- 下游依赖。
- rebalance。
- 热点 partition。

### 大消息

大消息会影响：

- 网络。
- broker 内存。
- consumer fetch。
- 延迟。

建议不要把大文件直接放 Kafka。可以把文件放对象存储，Kafka 只传引用。

## Go 服务配置建议

不要把安全配置写死在代码里。

推荐环境变量：

```text
KAFKA_BROKERS=
KAFKA_SECURITY_PROTOCOL=
KAFKA_SASL_MECHANISM=
KAFKA_USERNAME=
KAFKA_PASSWORD=
KAFKA_TLS_CA_FILE=
```

## 本节练习

1. 为订单服务列出它需要的 Kafka ACL。
2. 为通知服务列出它需要的 Kafka ACL。
3. 思考：为什么生产环境不能所有服务共享一个 Kafka 用户？
4. 写一份 consumer lag 飙升排查清单。

