# Docker Compose 实验环境

## 1. 目录结构

```text
prometheus-lab/
├── docker-compose.yml
├── prometheus/
│   ├── prometheus.yml
│   └── rules/
├── alertmanager/
│   └── alertmanager.yml
└── app/
```

本节先启动 Prometheus 和 Grafana；Alertmanager 在告警阶段再配置。

## 2. Compose 配置

创建 `docker-compose.yml`：

```yaml
services:
  prometheus:
    image: prom/prometheus:v3.0.0
    container_name: prom-lab-prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=7d
      - --web.enable-lifecycle
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    depends_on:
      - app
    networks:
      - lab

  grafana:
    image: grafana/grafana:11.0.0
    container_name: prom-lab-grafana
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: change-me-locally
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - prometheus
    networks:
      - lab

  app:
    image: hashicorp/http-echo:1.0
    container_name: prom-lab-app
    command:
      - -listen=:8080
      - -text=hello
    expose:
      - "8080"
    networks:
      - lab

volumes:
  prometheus-data:
  grafana-data:

networks:
  lab:
```

版本号只是教学示例。实际使用前确认镜像 tag 存在，并把 Prometheus、Grafana 和 Alertmanager 的 tag 统一记录到项目 README；生产环境不要使用 `latest`。

## 3. Prometheus 配置

创建 `prometheus/prometheus.yml`：

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["prometheus:9090"]

  - job_name: demo-app
    metrics_path: /metrics
    static_configs:
      - targets: ["app:8080"]
```

注意：`hashicorp/http-echo` 只返回普通文本，不会暴露合法的 Prometheus 指标；这个配置故意用于练习“抓取成功但解析失败”。下一步需要把 `app` 换成真正暴露 `/metrics` 的 Go 服务，或者先把它当作错误实验 target。

## 4. 启动和检查

```bash
docker compose up -d
docker compose ps
docker compose logs --tail=100 prometheus
curl http://localhost:9090/-/ready
curl http://localhost:9090/api/v1/targets
```

访问：

- Prometheus：`http://localhost:9090`
- Grafana：`http://localhost:3000`
- Grafana 初始账号是实验配置中的 `admin` / `change-me-locally`，首次登录后立即修改。

停止但保留数据：

```bash
docker compose stop
docker compose start
```

停止并删除容器：

```bash
docker compose down
```

删除实验数据：

```bash
docker compose down -v
```

`down -v` 会删除 Prometheus 和 Grafana 数据卷，只在你明确需要清空实验时使用。

## 5. 用 Go 服务替换 demo-app

把 app 服务改为本地构建的 Go 服务：

```yaml
  app:
    build:
      context: ./app
    expose:
      - "8080"
    networks:
      - lab
```

Prometheus target 仍使用 `app:8080`，因为 Compose 服务名在同一个网络中可解析。不要写成 `localhost:8080`。

## 6. 配置热加载实验

配置中启用 `--web.enable-lifecycle` 后可以：

```bash
curl -X POST http://localhost:9090/-/reload
```

配置修改后先检查：

```bash
docker exec prom-lab-prometheus promtool check config /etc/prometheus/prometheus.yml
```

如果容器中的 promtool 路径或镜像行为不同，使用与你镜像版本匹配的独立 promtool。

## 7. 完成标准

- 能从零创建并销毁环境。
- 能说明两个容器之间为什么用服务名通信。
- 能解释 app target 的“连接成功但格式错误”。
- 能删除数据卷并确认旧查询数据消失。
