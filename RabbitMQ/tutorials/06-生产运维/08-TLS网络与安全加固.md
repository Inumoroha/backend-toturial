# 08. TLS、网络与安全加固

## 1. 安全目标

RabbitMQ 安全至少包含：

- 身份认证。
- 权限控制。
- 网络隔离。
- 传输加密。
- 管理后台保护。
- 密钥治理。

第 7 节已经学习权限，本节重点看网络和 TLS。

## 2. 不要把 RabbitMQ 暴露到公网

生产环境中，RabbitMQ 的端口不应直接暴露到公网：

```text
5672  AMQP
15672 Management UI
15692 Prometheus metrics
```

建议：

- 只允许内网访问。
- 使用安全组或防火墙限制来源。
- Management UI 放在 VPN、堡垒机或内网。
- Prometheus metrics 只允许监控系统访问。

## 3. 默认 guest 用户限制

RabbitMQ 默认 `guest/guest` 只适合本地学习。

生产环境建议：

- 删除或禁用默认 guest。
- 创建明确的应用账号。
- 使用强密码。
- 定期轮换。

查看用户：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl list_users
```

删除用户：

```powershell
docker exec -it rabbitmq-dev rabbitmqctl delete_user guest
```

学习环境不要随手删除，除非你知道自己在做什么。

## 4. TLS 的作用

TLS 用来加密客户端和 RabbitMQ 之间的传输。

适合：

- 跨主机通信。
- 跨网络区域通信。
- 对安全有要求的生产环境。

TLS 可以保护：

- 用户名密码。
- 消息内容。
- 管理 API 通信。

## 5. AMQP TLS 端口

常见端口：

```text
5672  普通 AMQP
5671  AMQP over TLS
```

实际端口可配置。

生产环境可以只开放 TLS 端口。

## 6. TLS 配置方向

RabbitMQ TLS 通常需要：

- CA 证书。
- 服务端证书。
- 服务端私钥。
- RabbitMQ 配置文件。
- 客户端信任 CA。

配置示意：

```ini
listeners.ssl.default = 5671
ssl_options.cacertfile = /path/to/ca_certificate.pem
ssl_options.certfile   = /path/to/server_certificate.pem
ssl_options.keyfile    = /path/to/server_key.pem
ssl_options.verify     = verify_peer
ssl_options.fail_if_no_peer_cert = false
```

具体配置要以你的部署方式和证书策略为准。

## 7. Go 客户端 TLS 方向

Go 客户端连接 TLS 时，不再是：

```text
amqp://user:pass@host:5672/
```

而是：

```text
amqps://user:pass@host:5671/
```

如果需要自定义证书校验，可以使用客户端支持的 TLS 配置方式。

学习阶段先理解方向，生产落地时要按公司证书体系实现。

## 8. Management UI 安全

Management UI 风险很高。

建议：

- 不暴露公网。
- 使用 HTTPS。
- 使用强密码。
- 用户分级授权。
- 限制访问来源 IP。
- 开启审计或至少保留访问日志。

不要把 `15672` 直接暴露给互联网。

## 9. Metrics 安全

Prometheus metrics 可能暴露：

- 队列名称。
- vhost。
- 节点状态。
- 业务拓扑。

所以 `15692` 也应限制访问。

只允许 Prometheus 访问。

## 10. 安全检查清单

- [ ] 没有使用默认 `guest/guest`。
- [ ] 应用账号使用最小权限。
- [ ] RabbitMQ 端口不暴露公网。
- [ ] Management UI 受网络限制。
- [ ] Prometheus metrics 受网络限制。
- [ ] 密码不写在代码里。
- [ ] 需要时启用 TLS。
- [ ] 定期审计用户和权限。
- [ ] 离职和服务下线时回收账号。

## 11. 本节小结

RabbitMQ 安全不是单个 TLS 配置，而是一组措施：

```text
网络隔离
强认证
最小权限
传输加密
后台保护
密钥治理
```

下一节整理生产部署检查清单。

