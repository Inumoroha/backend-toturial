# 02. Management UI 与诊断命令

## 1. Management UI 是第一观察入口

浏览器打开：

```text
http://localhost:15672
```

登录：

```text
go_learner / go_learner_pwd
```

生产环境中，Management UI 不应该暴露到公网。

## 2. Overview 页面看什么

Overview 是总览页。

重点看：

- Queued messages。
- Message rates。
- Global counts。
- Nodes。
- Memory。
- Disk free。
- File descriptors。
- Socket descriptors。

你要先判断：

```text
消息是否在堆积？
发布和消费速率是否平衡？
资源水位是否接近限制？
```

## 3. Queues 页面看什么

队列页面是排查业务问题最常用的页面。

重点字段：

| 字段 | 含义 |
| --- | --- |
| Ready | 等待消费的消息 |
| Unacked | 已投递但未 ack 的消息 |
| Total | Ready + Unacked |
| Consumers | 消费者数量 |
| Incoming | 进入队列速率 |
| Deliver / Get | 投递给消费者速率 |
| Ack | ack 速率 |

常见判断：

```text
Ready 持续增长 -> 消费跟不上或没有消费者。
Unacked 持续增长 -> 消费者拿到消息但处理慢或不 ack。
Consumers 为 0 -> 消费者掉线或没有部署。
```

## 4. Connections 页面看什么

Connection 是客户端 TCP 连接。

重点看：

- 连接数量。
- 连接来源 IP。
- 用户名。
- vhost。
- state。
- channels 数量。

异常情况：

```text
连接数快速增长
同一个服务创建大量连接
短连接频繁出现
```

这通常说明客户端连接复用有问题。

## 5. Channels 页面看什么

Channel 是 connection 上的 AMQP 信道。

重点看：

- channel 数量。
- prefetch。
- unacked。
- consumer count。

异常情况：

```text
channel 数量持续增长但不下降
```

可能是应用泄漏 channel。

## 6. Exchanges 页面看什么

重点看：

- exchange 类型。
- durable。
- bindings。
- message rates。

排查“发布成功但队列没消息”时，看：

```text
exchange 是否存在
binding 是否正确
routing key 是否匹配
```

## 7. Admin 页面看什么

Admin 页面用于管理：

- users。
- vhosts。
- permissions。
- policies。
- limits。

生产环境重点：

```text
不要所有应用共用管理员账号。
不同业务尽量使用不同 vhost 或至少不同 user。
权限要按 configure/write/read 收敛。
```

## 8. rabbitmqctl 常用命令

查看队列：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers
```

查看 exchange：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_exchanges name type durable auto_delete
```

查看 binding：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_bindings
```

查看连接：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_connections name user vhost state channels
```

查看 channel：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_channels connection number user vhost prefetch_count messages_unacknowledged
```

查看消费者：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_consumers
```

## 9. rabbitmq-diagnostics 常用命令

查看节点状态：

```powershell
docker exec -it rabbitmq-dev rabbitmq-diagnostics status
```

检查节点是否运行：

```powershell
docker exec -it rabbitmq-dev rabbitmq-diagnostics check_running
```

ping 节点：

```powershell
docker exec -it rabbitmq-dev rabbitmq-diagnostics ping
```

检查本地告警：

```powershell
docker exec -it rabbitmq-dev rabbitmq-diagnostics check_local_alarms
```

查看内存分布：

```powershell
docker exec -it rabbitmq-dev rabbitmq-diagnostics memory_breakdown
```

## 10. Management HTTP API

Management Plugin 提供 HTTP API。

查看 overview：

```powershell
curl.exe -u go_learner:go_learner_pwd http://localhost:15672/api/overview
```

查看队列：

```powershell
curl.exe -u go_learner:go_learner_pwd http://localhost:15672/api/queues
```

PowerShell 中建议使用 `curl.exe`，避免和 PowerShell alias 混淆。

## 11. 本节小结

排查 RabbitMQ 时，常用入口：

```text
Management UI
rabbitmqctl
rabbitmq-diagnostics
Management HTTP API
```

下一节学习核心监控指标。

