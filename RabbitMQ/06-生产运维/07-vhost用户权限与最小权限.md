# 07. vhost、用户权限与最小权限

## 1. 为什么要做权限隔离

学习环境可以一个账号走天下。

生产环境不可以。

如果所有服务都使用管理员账号：

- 某个服务可能误删 exchange。
- 某个服务可能读取不该读的队列。
- 账号泄露影响全部业务。
- 无法审计是谁做了什么。

生产环境要遵守：

```text
最小权限原则。
```

## 2. vhost 的作用

vhost 是逻辑隔离空间。

可以按环境隔离：

```text
/dev
/test
/prod
```

也可以按业务隔离：

```text
/order
/notification
/payment
```

同名 exchange 和 queue 在不同 vhost 中互不影响。

## 3. 用户规划

不要所有应用共用一个用户。

建议：

```text
order_publisher
order_consumer_inventory
order_consumer_points
notification_worker
outbox_worker
monitoring_reader
```

不同服务使用不同账号，权限更容易收敛。

## 4. RabbitMQ 权限三元组

RabbitMQ 权限分三类：

```text
configure
write
read
```

| 权限 | 作用 |
| --- | --- |
| configure | 声明、修改 exchange、queue、binding |
| write | 发布消息 |
| read | 消费消息 |

设置命令：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl set_permissions -p / user "configure-regex" "write-regex" "read-regex"
```

## 5. 示例：生产者账号

订单服务只需要发布订单事件。

它可能需要：

```text
write: order.events
read: 无
configure: 视团队策略
```

示例：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl add_user order_publisher strong_password
docker exec -it rabbitmq-dev rabbitmqctl set_permissions -p / order_publisher "^$" "^order\\.events$" "^$"
```

含义：

- 不能配置资源。
- 只能写 `order.events`。
- 不能读队列。

如果应用启动时需要自动声明 exchange，configure 需要放开对应资源。

## 6. 示例：消费者账号

库存消费者只读自己的队列：

```text
read: order.created.inventory.queue
write: 可能需要写 retry exchange 或发布后续事件
```

示例：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl add_user inventory_worker strong_password
docker exec -it rabbitmq-dev rabbitmqctl set_permissions -p / inventory_worker "^$" "^(inventory\\.events|order\\.retry\\.exchange)$" "^order\\.created\\.inventory\\.queue$"
```

实际正则要根据你的 exchange/queue 命名调整。

## 7. 管理员和监控账号

管理员账号：

```text
只给少数人或运维系统。
```

监控账号可以只读：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl add_user monitoring_reader strong_password
docker exec -it rabbitmq-dev rabbitmqctl set_user_tags monitoring_reader monitoring
```

具体权限按监控需要配置。

## 8. 不要在代码里硬编码密码

不推荐：

```go
const rabbitURL = "amqp://user:password@host:5672/"
```

推荐从配置读取：

```text
RABBITMQ_URL
```

生产环境使用：

- Kubernetes Secret。
- 环境变量。
- 配置中心。
- 密钥管理系统。

## 9. 定期审计

定期检查：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_users
docker exec -it rabbitmq-dev rabbitmqctl list_vhosts
docker exec -it rabbitmq-dev rabbitmqctl list_permissions -p /
```

重点看：

- 是否有不用的账号。
- 是否有过大的权限。
- 是否有弱密码。
- 是否有应用使用管理员账号。

## 10. 本节小结

生产权限建议：

- 使用 vhost 隔离环境或业务。
- 每个应用独立 user。
- configure/write/read 按需授权。
- 不要共用管理员账号。
- 密码不要硬编码。
- 定期审计权限。

下一节学习 TLS、网络和安全加固。

