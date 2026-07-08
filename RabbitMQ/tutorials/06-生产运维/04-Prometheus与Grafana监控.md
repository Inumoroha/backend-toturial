# 04. Prometheus 与 Grafana 监控

## 1. 为什么需要 Prometheus

Management UI 适合临时查看，不适合长期监控和告警。

生产环境需要：

- 指标采集。
- 历史趋势。
- Dashboard。
- 告警规则。

常见组合：

```text
RabbitMQ Prometheus Plugin -> Prometheus -> Grafana -> Alertmanager
```

## 2. RabbitMQ Prometheus Plugin

RabbitMQ 提供 Prometheus 插件。

在容器里启用：

```powershell
docker exec -it rabbitmq-dev rabbitmq-plugins enable rabbitmq_prometheus
```

默认指标端口通常是：

```text
15692
```

如果当前容器没有映射该端口，你需要重建容器或在 Compose 中添加端口映射。

示例：

```yaml
ports:
  - "5672:5672"
  - "15672:15672"
  - "15692:15692"
```

## 3. 检查 metrics

启用插件并映射端口后，访问：

```text
http://localhost:15692/metrics
```

PowerShell：

```powershell
curl.exe http://localhost:15692/metrics
```

如果能看到大量 `rabbitmq_` 开头的指标，说明采集端点可用。

## 4. Prometheus 配置示例

`prometheus.yml`：

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: rabbitmq
    static_configs:
      - targets:
          - rabbitmq:15692
```

如果 Prometheus 在宿主机运行：

```yaml
targets:
  - localhost:15692
```

## 5. Docker Compose 示例

```yaml
services:
  rabbitmq:
    image: rabbitmq:4-management
    container_name: rabbitmq-monitoring
    ports:
      - "5672:5672"
      - "15672:15672"
      - "15692:15692"
    environment:
      RABBITMQ_DEFAULT_USER: go_learner
      RABBITMQ_DEFAULT_PASS: go_learner_pwd

  prometheus:
    image: prom/prometheus
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "3000:3000"
```

启动：

```powershell
docker compose up -d
```

## 6. Grafana 看板建议

Dashboard 至少包含：

- 节点内存。
- 磁盘剩余。
- 连接数。
- channel 数。
- 队列 ready。
- 队列 unacked。
- publish rate。
- deliver rate。
- ack rate。
- consumer 数。
- DLQ 消息数。

按业务分组：

```text
订单队列
通知队列
重试队列
死信队列
Outbox 指标
```

不要只放 RabbitMQ 全局指标。

## 7. 监控粒度注意

如果队列很多，逐队列指标会增加 Prometheus 压力。

生产中要注意：

- 队列数量。
- 指标 cardinality。
- scrape interval。
- 保留时间。

不要随意创建大量动态队列并全部高频监控。

## 8. 应用侧指标

Go 应用也应该暴露指标：

- 消费成功数。
- 消费失败数。
- 处理耗时。
- 重试次数。
- DLQ 发布数。
- Outbox pending 数。
- Outbox publish 成功/失败数。

RabbitMQ 指标只能说明 broker 状态。

应用指标才能说明业务处理质量。

## 9. 本节小结

生产监控推荐链路：

```text
rabbitmq_prometheus -> Prometheus -> Grafana -> Alertmanager
```

你要同时监控：

- RabbitMQ 节点。
- RabbitMQ 队列。
- 应用消费者。
- Outbox。
- DLQ。

下一节学习告警规则设计。

