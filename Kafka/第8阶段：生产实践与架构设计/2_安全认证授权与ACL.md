# 2. 安全认证授权与 ACL

本节目标：理解 Kafka 生产环境中的 TLS、SASL、ACL，并知道 Go 服务应该如何配置访问权限。

---

## 一、为什么需要安全

生产 Kafka 里可能有：

- 订单事件。
- 支付事件。
- 用户信息。
- 行为日志。

不能让所有服务随便读写所有 topic。

---

## 二、TLS

TLS 用于加密通信。

解决：

```text
客户端和 broker 之间数据被窃听
```

Go 客户端需要配置 CA、证书和 broker 地址。

---

## 三、SASL

SASL 用于认证。

常见机制：

```text
PLAIN
SCRAM-SHA-256
SCRAM-SHA-512
OAUTHBEARER
```

Go 服务通过环境变量传入用户名密码：

```text
KAFKA_USERNAME
KAFKA_PASSWORD
```

---

## 四、ACL

ACL 用于授权。

示例：

```text
order-service:
  write order.created
  read none

inventory-service:
  read order.created
  write inventory.deducted

notification-service:
  read order.created,payment.succeeded
```

最小权限原则：

```text
服务只拥有自己需要的 topic 权限
```

---

## 五、不要共享账号

错误：

```text
所有服务使用 kafka_app
```

问题：

- 无法审计。
- 无法区分权限。
- 一个服务泄漏影响全部。

推荐：

```text
order-service-user
inventory-service-user
notification-service-user
```

---

## 六、Go 配置

```text
KAFKA_BROKERS=
KAFKA_SECURITY_PROTOCOL=SASL_SSL
KAFKA_SASL_MECHANISM=SCRAM-SHA-512
KAFKA_USERNAME=
KAFKA_PASSWORD=
KAFKA_TLS_CA_FILE=
```

不要把密码写死进代码或 Git。

---

## 七、消息内容安全

不要随意放：

- 身份证。
- 完整手机号。
- 支付敏感信息。
- 密码。
- token。

必要时脱敏或只传业务 ID。

---

## 八、本节练习

1. 为 order-service 设计 ACL。
2. 为 inventory-service 设计 ACL。
3. 解释 TLS 和 SASL 的区别。
4. 说明为什么不能所有服务共享一个 Kafka 用户。

---

## 九、本节小结

- TLS 负责加密。
- SASL 负责认证。
- ACL 负责授权。
- 生产环境应按服务分配最小权限。
- Go 配置通过环境变量注入。
- 消息中不要放敏感字段。

---

## 十、ACL 设计表

建议把权限写成表格：

| 服务 | Topic | 权限 | 说明 |
| --- | --- | --- | --- |
| order-service | order.created | Write | 发布订单创建事件 |
| outbox-worker | order.created | Write | 如果由 worker 发布事件 |
| inventory-service | order.created | Read | 消费订单创建事件 |
| inventory-service | inventory.deducted | Write | 发布库存扣减成功事件 |
| notification-service | order.created | Read | 消费通知相关事件 |
| dlq-tool | order.created.dlq | Read | 查询和重放死信 |

注意：

```text
如果使用 outbox-worker 统一发 Kafka，order-service 本身可能不需要 Kafka Write 权限。
```

---

## 十一、Consumer Group 权限

Kafka ACL 不只涉及 topic，也涉及 consumer group。

例如：

```text
inventory-service 需要读 topic order.created
inventory-service 还需要使用 group inventory-service
```

权限设计要包含：

- Topic Read。
- Topic Write。
- Group Read。
- TransactionalId，如使用事务。

---

## 十二、Go 配置示例

配置结构：

```go
type KafkaSecurityConfig struct {
    SecurityProtocol string
    SASLMechanism    string
    Username         string
    Password         string
    TLSCAFile        string
}
```

环境变量：

```text
KAFKA_SECURITY_PROTOCOL=SASL_SSL
KAFKA_SASL_MECHANISM=SCRAM-SHA-512
KAFKA_USERNAME=inventory-service
KAFKA_PASSWORD=******
KAFKA_TLS_CA_FILE=/etc/certs/ca.pem
```

密码不要写入：

- Git。
- README 示例真实值。
- 镜像。
- 日志。

---

## 十三、敏感字段处理

订单事件里可以放：

```text
order_id
user_id
total_amount
```

谨慎放：

```text
手机号
地址
身份证
银行卡
支付 token
```

更好的方式：

```text
Kafka 传业务 ID，下游按权限查询详情。
```

---

## 十四、权限事故案例

错误设计：

```text
所有服务共用 kafka-admin 用户。
```

后果：

- notification-service 也能写 payment.succeeded。
- 某服务泄漏密码后所有 topic 暴露。
- 审计时看不出谁写入了异常消息。

正确设计：

```text
每个服务一个账号。
每个账号只给需要的 topic 和 group 权限。
```

---

## 十五、上线前安全检查

- [ ] 是否启用 TLS？
- [ ] 是否启用 SASL？
- [ ] 是否每个服务独立账号？
- [ ] 是否按最小权限配置 ACL？
- [ ] 是否避免敏感字段进入消息？
- [ ] 密码是否走密钥管理？
- [ ] 日志是否会打印密码或 token？

---

## 十四、ACL 最小权限示例

`order-service` 只生产订单事件：

```text
WRITE order.created
DESCRIBE order.created
```

`inventory-service` 只消费订单事件并生产库存结果：

```text
READ order.created
READ order.created.retry.1m
WRITE inventory.deducted
WRITE inventory.failed
WRITE order.created.dlq
DESCRIBE group inventory-service
```

不要给所有服务统一的超级账号。

---

## 十五、消息内容安全

Kafka 消息里不要放身份证号、银行卡号、明文手机号、访问 token、密码。

如果业务确实需要用户标识，优先放内部 `user_id`，并让下游按权限查询详细信息。

---

## 十六、日志安全

日志里也不要打印：

```text
SASL 密码
连接串密码
token
完整消息 payload 中的敏感字段
```

安全不是只看 Kafka 配置，也要看应用日志。

---

## 十七、ACL 文档模板

```markdown
# Kafka ACL 设计

## 服务账号

| 服务 | 账号 |
| --- | --- |
| order-service | kafka-order-service |
| inventory-service | kafka-inventory-service |

## Topic 权限

| 服务 | Topic | 权限 |
| --- | --- | --- |
| order-service | order.created | WRITE, DESCRIBE |
| inventory-service | order.created | READ, DESCRIBE |
| inventory-service | inventory.deducted | WRITE, DESCRIBE |

## Group 权限

| 服务 | Group | 权限 |
| --- | --- | --- |
| inventory-service | inventory-service | READ |
```

权限文档要能让运维或平台同学直接照着配置。

---

## 十八、上线前安全检查

- [ ] 每个服务独立账号。
- [ ] 不共用超级用户。
- [ ] producer 只有写权限。
- [ ] consumer 只有读对应 topic 和 group 的权限。
- [ ] 消息 payload 没有敏感字段。
- [ ] 日志不会打印认证信息。
- [ ] 密码不写入 Git。

---

## 十九、运维交付物

安全设计最后要交付给平台或运维：

```text
服务账号列表。
topic 权限列表。
group 权限列表。
证书或密码发放方式。
回收权限流程。
```

权限不是写在文档里就结束，还要能落到实际配置。

---

## 二十、最终验收

检查 order-service 是否只能写 `order.created`，不能读取无关 topic。

检查 inventory-service 是否只能读 `order.created`，不能写订单 topic。

---

## 二十一、ACL 评审表

建议把权限整理成表格：

| 服务 | Topic | 权限 | 说明 |
| --- | --- | --- | --- |
| order-service | `order.created` | Write | 只能发布订单创建事件 |
| inventory-service | `order.created` | Read | 只能消费订单创建事件 |
| inventory-service | `inventory.reserved` | Write | 发布库存预占结果 |
| ops-tool | `*.DLQ` | Read | 仅运维重放工具读取 |

ACL 的目标不是增加配置复杂度，而是减少服务误读、误写、误删关键 topic 的风险。
