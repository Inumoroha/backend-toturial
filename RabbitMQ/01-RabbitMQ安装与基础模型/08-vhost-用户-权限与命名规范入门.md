# 08. vhost、用户、权限与命名规范入门

## 1. 为什么要学 vhost 和权限

RabbitMQ 不只是本地玩具。真实项目中，通常会有：

- 开发环境
- 测试环境
- 预发环境
- 生产环境
- 多个业务系统
- 多个应用账号

如果所有人都使用同一个管理员账号、同一个默认空间，后期会很难管理。

所以你需要从一开始就理解：

```text
vhost 用来隔离资源
user 用来表示身份
permission 用来控制权限
```

## 2. 什么是 vhost

Virtual Host，简称 vhost，可以理解为 RabbitMQ 里的逻辑命名空间。

不同 vhost 里的资源彼此隔离：

```text
vhost: /dev
  exchange: order.events
  queue: order.created.queue

vhost: /prod
  exchange: order.events
  queue: order.created.queue
```

虽然名字一样，但因为属于不同 vhost，所以是不同资源。

## 3. 创建 vhost

在 UI 中进入：

```text
Admin -> Virtual Hosts -> Add a new virtual host
```

填写：

```text
Name: rabbitmq_learning
```

点击：

```text
Add virtual host
```

也可以使用命令：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl add_vhost rabbitmq_learning
```

查看 vhost：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_vhosts
```

## 4. 创建用户

在 UI 中进入：

```text
Admin -> Users -> Add a user
```

填写：

```text
Username: app_order
Password: app_order_pwd
Tags: management
```

点击：

```text
Add user
```

命令方式：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl add_user app_order app_order_pwd
docker exec -it rabbitmq-dev rabbitmqctl set_user_tags app_order management
```

说明：

- `management` tag 表示可以登录管理后台。
- 生产环境中，业务应用账号通常不一定需要管理后台权限。

## 5. 设置权限

RabbitMQ 权限通常包含三类：

```text
configure
write
read
```

含义：

| 权限 | 作用 |
| --- | --- |
| configure | 声明、修改 exchange/queue/binding |
| write | 发布消息 |
| read | 消费消息 |

给 `app_order` 设置 `rabbitmq_learning` vhost 权限：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl set_permissions -p rabbitmq_learning app_order ".*" ".*" ".*"
```

三个 `".*"` 分别表示：

```text
configure: .*
write: .*
read: .*
```

也就是允许配置、写入、读取所有资源。

学习阶段可以这样设置，生产环境应按最小权限原则收紧。

查看权限：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_permissions -p rabbitmq_learning
```

## 6. 连接字符串中的 vhost

AMQP 连接地址通常长这样：

```text
amqp://username:password@host:port/vhost
```

例如默认 vhost `/`：

```text
amqp://go_learner:go_learner_pwd@localhost:5672/
```

如果 vhost 是 `rabbitmq_learning`：

```text
amqp://app_order:app_order_pwd@localhost:5672/rabbitmq_learning
```

后续 Go 代码中会用到这种连接字符串。

## 7. 命名规范建议

RabbitMQ 命名要清晰，否则后期排查会很痛苦。

### 7.1 Exchange 命名

建议包含业务域和用途：

```text
order.events
user.events
notification.direct
task.direct
```

### 7.2 Queue 命名

建议体现消费者或业务用途：

```text
order.created.inventory.queue
order.paid.points.queue
user.registered.email.queue
notification.email.worker.queue
```

### 7.3 Routing Key 命名

建议使用业务事件风格：

```text
order.created
order.paid
order.cancelled
user.registered
notification.email.send
```

### 7.4 避免的命名

不要使用过于模糊的名字：

```text
test
queue1
mq
data
event
abc
```

这些名字短期方便，长期会让排查变得非常痛苦。

## 8. 推荐学习环境命名

本教程后续可以统一使用：

```text
vhost: rabbitmq_learning
user: app_order
exchange: learning.direct
queue: learning.hello.queue
routing key: hello
```

如果你还在默认 vhost `/` 里练习，也没问题。第 1 阶段重点是理解概念。

## 9. 本节小结

你应该能解释：

- vhost 是逻辑隔离空间。
- user 表示连接 RabbitMQ 的身份。
- permission 控制 configure、write、read。
- 命名规范会显著影响后期排查效率。
- Go 连接字符串里会包含用户名、密码、地址、端口和 vhost。

