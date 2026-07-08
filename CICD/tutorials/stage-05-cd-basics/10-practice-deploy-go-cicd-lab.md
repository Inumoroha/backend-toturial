# 10：实战，部署 go-cicd-lab

## 1. 本节目标

这一节把第 5 阶段落到 `go-cicd-lab`。

最终产出：

```text
deploy/compose/compose.yaml
deploy/compose/.env.example
deploy/compose/app.env.example
scripts/deploy.sh
scripts/smoke-test.sh
.github/actions/ssh-deploy/action.yml
.github/workflows/deploy.yml
docs/deployment.md
```

## 2. 创建 Compose 文件

创建：

```text
deploy/compose/compose.yaml
```

内容：

```yaml
services:
  api:
    image: ${APP_IMAGE}
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

  postgres:
    image: postgres:17-alpine
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
    restart: unless-stopped
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis-data:/data

volumes:
  postgres-data:
  redis-data:
```

## 3. 创建环境模板

`deploy/compose/.env.example`：

```text
APP_IMAGE=ghcr.io/your-name/go-cicd-lab:sha-example
POSTGRES_DB=go_cicd_lab
POSTGRES_USER=go_cicd_lab
POSTGRES_PASSWORD=change-me
```

`deploy/compose/app.env.example`：

```text
APP_ENV=staging
HTTP_ADDR=:8080
DATABASE_URL=postgres://go_cicd_lab:change-me@postgres:5432/go_cicd_lab?sslmode=disable
REDIS_ADDR=redis:6379
LOG_LEVEL=info
```

真实文件只放服务器：

```text
/opt/go-cicd-lab/.env
/opt/go-cicd-lab/app.env
```

## 4. 创建部署脚本

创建：

```text
scripts/deploy.sh
```

内容：

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

if [ -f .current_image ]; then
  cat .current_image > .previous_image
fi

if grep -q '^APP_IMAGE=' .env; then
  sed -i "s|^APP_IMAGE=.*|APP_IMAGE=$IMAGE|" .env
else
  echo "APP_IMAGE=$IMAGE" >> .env
fi

docker compose pull api
docker compose up -d --remove-orphans api

deadline=$((SECONDS + TIMEOUT_SECONDS))
until curl -fsS "$HEALTH_URL" >/dev/null && curl -fsS "$READY_URL" >/dev/null; do
  if [ "$SECONDS" -ge "$deadline" ]; then
    docker compose ps >&2 || true
    docker compose logs --tail=100 api >&2 || true
    exit 1
  fi
  sleep 3
done

echo "$IMAGE" > .current_image
date -u +"%Y-%m-%dT%H:%M:%SZ $IMAGE" >> releases/deployments.log
echo "Deployment succeeded: $IMAGE"
```

## 5. 创建冒烟测试

创建：

```text
scripts/smoke-test.sh
```

内容：

```bash
#!/usr/bin/env bash
set -euo pipefail

BASE_URL="${BASE_URL:?BASE_URL is required}"

curl -fsS "$BASE_URL/healthz" >/dev/null
curl -fsS "$BASE_URL/readyz" >/dev/null

echo "Smoke tests passed for $BASE_URL"
```

## 6. 服务器初次部署

在服务器上：

```bash
sudo mkdir -p /opt/go-cicd-lab
sudo chown -R deploy:deploy /opt/go-cicd-lab
```

复制文件：

```text
compose.yaml -> /opt/go-cicd-lab/compose.yaml
scripts/deploy.sh -> /opt/go-cicd-lab/deploy.sh
```

创建真实配置：

```text
/opt/go-cicd-lab/.env
/opt/go-cicd-lab/app.env
```

设置权限：

```bash
chmod 600 /opt/go-cicd-lab/.env /opt/go-cicd-lab/app.env
chmod +x /opt/go-cicd-lab/deploy.sh
```

手动测试：

```bash
cd /opt/go-cicd-lab
IMAGE=ghcr.io/your-name/go-cicd-lab:sha-example ./deploy.sh
```

## 7. 创建 GitHub composite action

创建：

```text
.github/actions/ssh-deploy/action.yml
```

内容：

```yaml
name: ssh-deploy
description: Deploy over SSH with Docker Compose

inputs:
  host:
    required: true
  user:
    required: true
  ssh_key:
    required: true
  app_dir:
    required: true
  image:
    required: true
  health_url:
    required: true
  ready_url:
    required: true

runs:
  using: composite
  steps:
    - name: Configure SSH
      shell: bash
      run: |
        mkdir -p ~/.ssh
        chmod 700 ~/.ssh
        printf '%s\n' "${{ inputs.ssh_key }}" > ~/.ssh/deploy_key
        chmod 600 ~/.ssh/deploy_key
        ssh-keyscan -H "${{ inputs.host }}" >> ~/.ssh/known_hosts

    - name: Deploy
      shell: bash
      run: |
        ssh -i ~/.ssh/deploy_key "${{ inputs.user }}@${{ inputs.host }}" \
          "cd '${{ inputs.app_dir }}' && \
           APP_DIR='${{ inputs.app_dir }}' \
           IMAGE='${{ inputs.image }}' \
           HEALTH_URL='${{ inputs.health_url }}' \
           READY_URL='${{ inputs.ready_url }}' \
           ./deploy.sh"
```

## 8. 创建 deploy workflow

创建：

```text
.github/workflows/deploy.yml
```

内容：

```yaml
name: deploy

on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Target environment"
        required: true
        type: choice
        options:
          - staging
          - production
      image:
        description: "Image to deploy"
        required: true
        type: string
  push:
    branches: [main]
    tags:
      - "v*"

permissions:
  contents: read

jobs:
  staging:
    name: staging
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: ${{ secrets.HEALTH_URL }}
    concurrency: staging
    steps:
      - uses: actions/checkout@v6
      - name: Set image
        run: |
          image_repository="${GITHUB_REPOSITORY,,}"
          short_sha="${GITHUB_SHA::7}"
          echo "IMAGE=ghcr.io/${image_repository}:sha-${short_sha}" >> "$GITHUB_ENV"
      - uses: ./.github/actions/ssh-deploy
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          user: ${{ secrets.DEPLOY_USER }}
          ssh_key: ${{ secrets.DEPLOY_SSH_KEY }}
          app_dir: ${{ secrets.APP_DIR }}
          image: ${{ env.IMAGE }}
          health_url: ${{ secrets.HEALTH_URL }}
          ready_url: ${{ secrets.READY_URL }}

  production:
    name: production
    if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    environment:
      name: production
      url: ${{ secrets.HEALTH_URL }}
    concurrency: production
    steps:
      - uses: actions/checkout@v6
      - name: Set image
        run: |
          image_repository="${GITHUB_REPOSITORY,,}"
          echo "IMAGE=ghcr.io/${image_repository}:${GITHUB_REF_NAME}" >> "$GITHUB_ENV"
      - uses: ./.github/actions/ssh-deploy
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          user: ${{ secrets.DEPLOY_USER }}
          ssh_key: ${{ secrets.DEPLOY_SSH_KEY }}
          app_dir: ${{ secrets.APP_DIR }}
          image: ${{ env.IMAGE }}
          health_url: ${{ secrets.HEALTH_URL }}
          ready_url: ${{ secrets.READY_URL }}

  manual:
    name: manual
    if: github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    environment:
      name: ${{ inputs.environment }}
      url: ${{ secrets.HEALTH_URL }}
    concurrency: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v6
      - uses: ./.github/actions/ssh-deploy
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          user: ${{ secrets.DEPLOY_USER }}
          ssh_key: ${{ secrets.DEPLOY_SSH_KEY }}
          app_dir: ${{ secrets.APP_DIR }}
          image: ${{ inputs.image }}
          health_url: ${{ secrets.HEALTH_URL }}
          ready_url: ${{ secrets.READY_URL }}
```

注意：这里假设第 4 阶段 image workflow 会生成短 SHA tag，例如 `sha-8f3a2c1`，以及 release tag，例如 `v0.1.0`。实际项目必须确认 image workflow 和 deploy workflow 的 tag 规则完全一致。

## 9. 创建部署文档

创建：

```text
docs/deployment.md
```

内容模板：

````markdown
# 部署说明

## 环境

- staging：main 自动部署。
- production：tag `v*` 触发，GitHub environment 审批后部署。

## 服务器目录

```text
/opt/go-cicd-lab/
  compose.yaml
  .env
  app.env
  deploy.sh
  releases/
```

## 手动部署

```bash
cd /opt/go-cicd-lab
IMAGE=ghcr.io/<owner>/<repo>:sha-xxxx ./deploy.sh
```

## 手动回滚

```bash
cd /opt/go-cicd-lab
IMAGE=<previous-image> ./deploy.sh
```

## 验证

```bash
curl -f http://127.0.0.1:8080/healthz
curl -f http://127.0.0.1:8080/readyz
docker compose ps
docker compose logs --tail=100 api
```

## 部署记录

```bash
cat /opt/go-cicd-lab/.current_image
cat /opt/go-cicd-lab/.previous_image
cat /opt/go-cicd-lab/releases/deployments.log
```
````

## 10. 提交 PR

```bash
git switch -c cd/add-compose-vm-deployment
git add deploy/compose scripts .github/actions/ssh-deploy .github/workflows/deploy.yml docs/deployment.md
git commit -m "ci: add compose deployment workflow"
git push -u origin cd/add-compose-vm-deployment
```

## 11. 验收

你完成后应该能做到：

- 本地 Compose 能启动。
- VM 上能手动部署镜像。
- GitHub Actions 能手动部署 staging。
- main 合并后自动部署 staging。
- tag `v*` 触发 production 审批。
- 部署失败时能看到日志。
- 能手动回滚到指定镜像。

## 12. 本节小结

你现在已经把 `go-cicd-lab` 从“镜像可发布”推进到“环境可部署”。

这条链路虽然简单，但包含了真实 CD 的关键：环境、密钥、部署脚本、健康检查、审批和回滚。
