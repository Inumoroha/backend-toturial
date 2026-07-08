# 05. Docker Compose 本地环境

## 1. 本节目标

搭建项目本地依赖：

- RabbitMQ
- MySQL 或 PostgreSQL
- 可选 Prometheus
- 可选 Grafana

学习阶段建议先使用 MySQL。

## 2. docker-compose.yml

创建：

```text
deployments/docker-compose.yml
```

内容：

```yaml
services:
  rabbitmq:
    image: rabbitmq:4-management
    container_name: ecommerce-rabbitmq
    hostname: ecommerce-rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
      - "15692:15692"
    environment:
      RABBITMQ_DEFAULT_USER: go_learner
      RABBITMQ_DEFAULT_PASS: go_learner_pwd
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq

  mysql:
    image: mysql:8
    container_name: ecommerce-mysql
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: ecommerce_lab
      MYSQL_USER: app
      MYSQL_PASSWORD: app_pwd
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  rabbitmq_data:
  mysql_data:
```

## 3. 启动环境

在项目根目录执行：

```powershell
docker compose -f .\deployments\docker-compose.yml up -d
```

查看容器：

```powershell
docker ps
```

RabbitMQ UI：

```text
http://localhost:15672
```

账号：

```text
go_learner / go_learner_pwd
```

## 4. 检查 RabbitMQ

```powershell
docker exec -it ecommerce-rabbitmq rabbitmq-diagnostics status
docker exec -it ecommerce-rabbitmq rabbitmqctl list_queues
```

启用 Prometheus 插件：

```powershell
docker exec -it ecommerce-rabbitmq rabbitmq-plugins enable rabbitmq_prometheus
```

检查 metrics：

```powershell
curl.exe http://localhost:15692/metrics
```

## 5. 检查 MySQL

进入 MySQL：

```powershell
docker exec -it ecommerce-mysql mysql -uapp -papp_pwd ecommerce_lab
```

查看数据库：

```sql
SHOW TABLES;
```

## 6. 环境变量示例

创建 `.env.example`：

```text
APP_ENV=dev
HTTP_ADDR=:8080
DATABASE_DSN=app:app_pwd@tcp(localhost:3306)/ecommerce_lab?parseTime=true
RABBITMQ_URL=amqp://go_learner:go_learner_pwd@localhost:5672/
OUTBOX_BATCH_SIZE=100
OUTBOX_POLL_INTERVAL=1s
MQ_PREFETCH=5
```

不要把真实生产密码提交到仓库。

## 7. 停止环境

```powershell
docker compose -f .\deployments\docker-compose.yml down
```

如果要删除数据卷：

```powershell
docker compose -f .\deployments\docker-compose.yml down -v
```

注意：`-v` 会删除数据库和 RabbitMQ 数据。

## 8. 本节小结

本地环境要做到：

- 一条命令启动。
- 依赖明确。
- 数据持久化。
- RabbitMQ UI 可访问。
- 数据库可连接。

下一节设计数据库模型和迁移脚本。

