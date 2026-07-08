# 13. 项目 README、简历与面试表达

## 1. README 为什么重要

一个项目能不能写进简历，不只看代码。

面试官通常会先看：

- 项目解决什么问题。
- 架构是否清晰。
- 如何启动。
- 如何测试。
- 是否有可靠性设计。
- 是否能解释取舍。

所以 README 是项目的门面。

## 2. README 推荐结构

```text
# rabbitmq-go-ecommerce-lab

## 项目简介
## 技术栈
## 核心能力
## 系统架构
## RabbitMQ 拓扑
## 数据库设计
## 本地启动
## API 示例
## 可靠性设计
## 故障演练
## 测试
## 项目取舍
## 后续优化
```

## 3. 项目简介示例

```text
这是一个基于 Go 和 RabbitMQ 的电商异步订单系统，重点演示订单事件驱动、Transactional Outbox、可靠消息发布、幂等消费者、失败重试、死信队列和订单超时取消等后端工程能力。
```

## 4. 技术栈示例

```text
Go
RabbitMQ
MySQL
Docker Compose
amqp091-go
chi/gin
Prometheus 可选
Grafana 可选
```

## 5. 核心能力示例

```text
- 使用 RabbitMQ topic exchange 分发订单事件。
- 使用 Transactional Outbox 保证订单事务和消息发布一致。
- 使用 publisher confirm 和 mandatory publish 保证生产端可靠性。
- 消费者使用 manual ack、prefetch、retry queue 和 DLQ。
- 使用业务唯一键和数据库唯一索引保证消费者幂等。
- 使用 TTL + DLX 实现订单超时取消。
- 提供故障演练和集成测试。
```

## 6. 本地启动说明

README 中要写清楚：

```powershell
docker compose -f .\deployments\docker-compose.yml up -d
go run .\cmd\api
go run .\cmd\outbox-worker
go run .\cmd\inventory-worker
go run .\cmd\points-worker
go run .\cmd\notification-worker
go run .\cmd\timeout-worker
```

还要写：

```text
RabbitMQ UI: http://localhost:15672
MySQL: localhost:3306
```

## 7. API 示例

创建订单：

```powershell
curl.exe -X POST http://localhost:8080/api/orders -H "content-type: application/json" -d "{\"user_id\":1001,\"items\":[{\"sku_id\":2001,\"quantity\":2}]}"
```

支付订单：

```powershell
curl.exe -X POST http://localhost:8080/api/orders/1/pay
```

查询订单：

```powershell
curl.exe http://localhost:8080/api/orders/1
```

## 8. 简历描述

可以写：

```text
Go + RabbitMQ 电商异步订单系统：设计并实现订单创建、支付、库存预占、积分发放、通知发送和订单超时取消等异步流程。使用 Transactional Outbox 保证数据库事务与消息发布一致，结合 publisher confirm、manual ack、retry queue、DLQ 和业务幂等键构建可靠消息链路，并通过 Docker Compose、集成测试和故障演练验证消息堆积、重复消费和消费者崩溃等场景。
```

## 9. 面试讲解稿

可以这样讲：

```text
这个项目的核心是用 RabbitMQ 做订单事件驱动。订单 API 不直接发布消息，而是在同一个数据库事务中写 orders 和 outbox_messages。Outbox Worker 扫描 pending 消息，通过 publisher confirm 发布到 RabbitMQ。消费者使用 manual ack，处理失败进入 retry queue，超过次数进入 DLQ。为了应对至少一次投递，积分、库存、通知消费者都使用业务唯一键和数据库唯一索引做幂等。订单超时取消使用 TTL + DLX 实现延迟检查，并通过状态机条件更新避免取消已支付订单。
```

## 10. 常见追问准备

### 为什么用 Outbox？

```text
为了解决数据库事务成功但消息发布失败的问题。业务表和 outbox 表同事务写入，Outbox Worker 后续可靠发布。
```

### 为什么消费者要幂等？

```text
RabbitMQ 可靠消费通常是至少一次投递，消费者处理成功但 ack 前崩溃会导致消息重投，所以必须用业务唯一键保证重复消息不会重复执行业务副作用。
```

### 消息堆积怎么排查？

```text
先看 ready、unacked、consumers、publish rate 和 ack rate。ready 高通常是消费者不足或处理慢；unacked 高通常是消费者卡住、prefetch 太大或没有 ack；consumers 为 0 说明消费者掉线。
```

## 11. 本节小结

项目最终要能：

- 看得懂。
- 跑得起来。
- 测得出来。
- 讲得清楚。

下一节完成第 8 阶段最终交付清单。

