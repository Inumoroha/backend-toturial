# 05：通过 SSH 执行部署脚本

## 1. 本节目标

这一节编写 `deploy.sh`。

它要做：

- 接收要部署的镜像。
- 记录当前版本。
- 更新 `.env` 中的 `APP_IMAGE`。
- 拉取新镜像。
- 用 Docker Compose 更新服务。
- 等待健康检查。
- 成功后记录新版本。
- 失败时提示回滚。

## 2. 部署脚本放在哪里

推荐仓库中保存模板：

```text
scripts/deploy.sh
```

服务器上放实际脚本：

```text
/opt/go-cicd-lab/deploy.sh
```

脚本应该提交到 Git，这样部署逻辑可审查、可追踪。

## 3. deploy.sh 示例

```bash
#!/usr/bin/env bash
set -euo pipefail

APP_DIR="${APP_DIR:-/opt/go-cicd-lab}"
IMAGE="${IMAGE:?IMAGE is required}"
HEALTH_URL="${HEALTH_URL:-http://127.0.0.1:8080/healthz}"
READY_URL="${READY_URL:-http://127.0.0.1:8080/readyz}"
TIMEOUT_SECONDS="${TIMEOUT_SECONDS:-90}"

cd "$APP_DIR"

mkdir -p releases

PREVIOUS_IMAGE=""
if [ -f .current_image ]; then
  PREVIOUS_IMAGE="$(cat .current_image)"
  echo "$PREVIOUS_IMAGE" > .previous_image
fi

echo "Deploying image: $IMAGE"
echo "Previous image: ${PREVIOUS_IMAGE:-none}"

if [ ! -f .env ]; then
  echo ".env not found in $APP_DIR" >&2
  exit 1
fi

if grep -q '^APP_IMAGE=' .env; then
  sed -i "s|^APP_IMAGE=.*|APP_IMAGE=$IMAGE|" .env
else
  echo "APP_IMAGE=$IMAGE" >> .env
fi

docker compose pull api
docker compose up -d --remove-orphans api

echo "Waiting for health checks..."
deadline=$((SECONDS + TIMEOUT_SECONDS))

until curl -fsS "$HEALTH_URL" >/dev/null && curl -fsS "$READY_URL" >/dev/null; do
  if [ "$SECONDS" -ge "$deadline" ]; then
    echo "Health check failed for $IMAGE" >&2
    echo "Container status:" >&2
    docker compose ps >&2 || true
    echo "Recent logs:" >&2
    docker compose logs --tail=100 api >&2 || true
    exit 1
  fi
  sleep 3
done

echo "$IMAGE" > .current_image
date -u +"%Y-%m-%dT%H:%M:%SZ $IMAGE" >> releases/deployments.log

echo "Deployment succeeded: $IMAGE"
```

## 4. 脚本参数

必需：

```bash
IMAGE=ghcr.io/your-name/go-cicd-lab:sha-abc123
```

可选：

```bash
APP_DIR=/opt/go-cicd-lab
HEALTH_URL=http://127.0.0.1:8080/healthz
READY_URL=http://127.0.0.1:8080/readyz
TIMEOUT_SECONDS=90
```

## 5. 手动执行部署

在服务器上：

```bash
cd /opt/go-cicd-lab
chmod +x deploy.sh
IMAGE=ghcr.io/your-name/go-cicd-lab:sha-abc123 ./deploy.sh
```

查看：

```bash
docker compose ps
docker compose logs -f api
cat .current_image
cat releases/deployments.log
```

## 6. 从本机通过 SSH 执行

```bash
ssh -i ./go-cicd-lab-deploy deploy@your-server-ip \
  'cd /opt/go-cicd-lab && IMAGE=ghcr.io/your-name/go-cicd-lab:sha-abc123 ./deploy.sh'
```

如果这条能从本机跑通，就可以放进 GitHub Actions。

## 7. 为什么脚本里要健康检查

没有健康检查时：

```text
docker compose up -d 成功
```

只代表容器启动命令执行了，不代表服务真的可用。

健康检查可以发现：

- 服务启动后立刻崩溃。
- 端口未监听。
- 数据库连接失败。
- ready 逻辑失败。

## 8. 为什么记录 current 和 previous

```text
.current_image
.previous_image
releases/deployments.log
```

它们用于：

- 知道当前部署版本。
- 快速找到上一个版本。
- 回滚。
- 审计。

## 9. 初版脚本的限制

这个脚本适合学习和小型环境。

它不是完整生产发布系统。

限制：

- 单机部署。
- 没有流量切换。
- 没有蓝绿或金丝雀。
- 数据库迁移需要额外设计。
- 回滚需要单独脚本或手动传入旧镜像。

第 6 阶段以后会学习 Kubernetes 和更成熟的发布模型。

## 10. 小练习

1. 在服务器上创建 `deploy.sh`。
2. 手动部署一个镜像。
3. 查看 `.current_image`。
4. 再部署另一个镜像。
5. 查看 `.previous_image`。
6. 故意传入不存在的 tag，观察失败日志。

## 11. 本节小结

你现在应该理解：

- 部署脚本应该接收明确镜像。
- `docker compose pull` 和 `docker compose up -d` 是单机部署核心动作。
- 部署脚本必须做健康检查。
- 部署历史和前一个版本是回滚基础。

