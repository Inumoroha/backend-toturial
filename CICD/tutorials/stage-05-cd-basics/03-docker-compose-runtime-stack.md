# 03：Docker Compose 运行栈

## 1. 本节目标

这一节用 Docker Compose 定义 Go 后端服务的运行环境。

学习目标：

- 定义 API 服务。
- 可选定义 PostgreSQL 和 Redis。
- 使用 named volumes 保存数据。
- 使用 healthcheck。
- 使用 `.env` 和 `env_file` 管理配置。

## 2. 为什么用 Docker Compose

Docker Compose 适合：

- 本地多容器开发。
- 单台 VM 入门部署。
- 小型内部服务。
- 学习 CD 流程。

它不是 Kubernetes 的替代品，但非常适合第 5 阶段理解部署动作。

## 3. 推荐目录

在服务器上建议：

```text
/opt/go-cicd-lab/
  compose.yaml
  .env
  app.env
  deploy.sh
  releases/
```

在仓库中建议：

```text
deploy/compose/compose.yaml
deploy/compose/app.env.example
scripts/deploy.sh
```

真实 `.env`、`app.env` 只放服务器或 secret manager，不提交。

## 4. compose.yaml 示例

```yaml
services:
  api:
    image: ${APP_IMAGE}
    container_name: go-cicd-lab-api
    restart: unless-stopped
    env_file:
      - ./app.env
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "/server", "healthcheck"]
      interval: 10s
      timeout: 3s
      retries: 6
      start_period: 20s

  postgres:
    image: postgres:17-alpine
    container_name: go-cicd-lab-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 3s
      retries: 10

  redis:
    image: redis:7-alpine
    container_name: go-cicd-lab-redis
    restart: unless-stopped
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis-data:/data

volumes:
  postgres-data:
  redis-data:
```

注意：`/server healthcheck` 需要你的 Go 程序支持该命令。如果还没实现，可以先用 HTTP healthcheck，见下一节。

## 5. HTTP healthcheck 版本

如果镜像中有 `wget` 或 `curl`，可以：

```yaml
healthcheck:
  test: ["CMD", "wget", "-qO-", "http://127.0.0.1:8080/healthz"]
```

但 distroless 镜像通常没有 `wget` 或 `curl`。

所以 Go 服务更推荐内置一个 CLI healthcheck，或者依靠部署脚本从容器外访问：

```bash
curl -f http://127.0.0.1:8080/healthz
```

初学阶段可以先在部署脚本中做外部 HTTP 检查。

## 6. .env 示例

`.env` 给 Compose 做变量替换：

```text
APP_IMAGE=ghcr.io/your-name/go-cicd-lab:sha-abc123
POSTGRES_DB=go_cicd_lab
POSTGRES_USER=go_cicd_lab
POSTGRES_PASSWORD=change-me
```

不要提交真实 `.env`。

可以提交：

```text
deploy/compose/.env.example
```

## 7. app.env 示例

`app.env` 给应用容器做环境变量：

```text
APP_ENV=staging
HTTP_ADDR=:8080
DATABASE_URL=postgres://go_cicd_lab:change-me@postgres:5432/go_cicd_lab?sslmode=disable
REDIS_ADDR=redis:6379
LOG_LEVEL=info
```

可以提交模板：

```text
deploy/compose/app.env.example
```

不要提交真实密码。

## 8. 常用 Compose 命令

启动：

```bash
docker compose up -d
```

拉取镜像：

```bash
docker compose pull
```

查看状态：

```bash
docker compose ps
```

查看日志：

```bash
docker compose logs -f api
```

重启单个服务：

```bash
docker compose up -d --no-deps api
```

停止并删除容器和网络：

```bash
docker compose down
```

注意：`docker compose down -v` 会删除 volumes，可能删除数据库数据。生产环境不要随便用。

## 9. 数据库是否应该也放 Compose

学习和小型 staging 可以放。

真实生产更推荐：

- 托管数据库。
- 独立数据库服务器。
- 完整备份、监控和恢复策略。

本阶段把 PostgreSQL 放 Compose 是为了学习完整运行栈，不代表所有生产都应这么做。

## 10. 小练习

1. 创建 `deploy/compose/compose.yaml`。
2. 创建 `.env.example` 和 `app.env.example`。
3. 本地复制模板：

```bash
cp deploy/compose/.env.example deploy/compose/.env
cp deploy/compose/app.env.example deploy/compose/app.env
```

4. 启动：

```bash
cd deploy/compose
docker compose up -d
docker compose ps
docker compose logs -f api
```

## 11. 本节小结

你现在应该理解：

- Compose 可以定义单机多容器运行栈。
- `.env` 用于 Compose 变量替换，`env_file` 用于应用环境变量。
- volumes 保存数据库和 Redis 数据。
- healthcheck 和部署脚本验证都很重要。
- 真实生产数据库通常需要更可靠的方案。

