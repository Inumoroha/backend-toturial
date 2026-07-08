# 06：GitHub Actions 部署 Workflow

## 1. 本节目标

这一节把手动 SSH 部署接入 GitHub Actions。

目标：

```text
main -> 自动部署 staging
tag v* -> 等待 production 审批 -> 部署 production
workflow_dispatch -> 手动选择环境和镜像
```

## 2. GitHub Environments 准备

在仓库设置中创建：

```text
staging
production
```

建议：

staging：

- 不需要审批。
- 只允许 main 分支部署，可选。

production：

- required reviewers。
- 可选 wait timer。
- 只允许 tag 或 main 部署。
- environment secrets 单独配置。

GitHub 文档说明，job 引用 environment 时，必须满足该 environment 的保护规则后，job 才能运行并访问 environment secrets。

## 3. Environment Secrets

在 `staging` environment 中配置：

```text
DEPLOY_HOST
DEPLOY_USER
DEPLOY_SSH_KEY
APP_DIR
HEALTH_URL
READY_URL
```

在 `production` environment 中也配置同名 secrets，但值不同。

这样 workflow 可以复用同一套变量名。

## 4. 创建 deploy workflow

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
  deploy-staging:
    name: deploy-staging
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: ${{ secrets.HEALTH_URL }}
    concurrency:
      group: staging
      cancel-in-progress: false
    steps:
      - name: Checkout
        uses: actions/checkout@v6

      - name: Set image
        run: |
          image_repository="${GITHUB_REPOSITORY,,}"
          short_sha="${GITHUB_SHA::7}"
          echo "IMAGE=ghcr.io/${image_repository}:sha-${short_sha}" >> "$GITHUB_ENV"

      - name: Deploy
        uses: ./.github/actions/ssh-deploy
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          user: ${{ secrets.DEPLOY_USER }}
          ssh_key: ${{ secrets.DEPLOY_SSH_KEY }}
          app_dir: ${{ secrets.APP_DIR }}
          image: ${{ env.IMAGE }}
          health_url: ${{ secrets.HEALTH_URL }}
          ready_url: ${{ secrets.READY_URL }}

  deploy-production:
    name: deploy-production
    if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    environment:
      name: production
      url: ${{ secrets.HEALTH_URL }}
    concurrency:
      group: production
      cancel-in-progress: false
    steps:
      - name: Checkout
        uses: actions/checkout@v6

      - name: Set image
        run: |
          image_repository="${GITHUB_REPOSITORY,,}"
          echo "IMAGE=ghcr.io/${image_repository}:${GITHUB_REF_NAME}" >> "$GITHUB_ENV"

      - name: Deploy
        uses: ./.github/actions/ssh-deploy
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          user: ${{ secrets.DEPLOY_USER }}
          ssh_key: ${{ secrets.DEPLOY_SSH_KEY }}
          app_dir: ${{ secrets.APP_DIR }}
          image: ${{ env.IMAGE }}
          health_url: ${{ secrets.HEALTH_URL }}
          ready_url: ${{ secrets.READY_URL }}

  deploy-manual:
    name: deploy-manual
    if: github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    environment:
      name: ${{ inputs.environment }}
      url: ${{ secrets.HEALTH_URL }}
    concurrency:
      group: ${{ inputs.environment }}
      cancel-in-progress: false
    steps:
      - name: Checkout
        uses: actions/checkout@v6

      - name: Deploy
        uses: ./.github/actions/ssh-deploy
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          user: ${{ secrets.DEPLOY_USER }}
          ssh_key: ${{ secrets.DEPLOY_SSH_KEY }}
          app_dir: ${{ secrets.APP_DIR }}
          image: ${{ inputs.image }}
          health_url: ${{ secrets.HEALTH_URL }}
          ready_url: ${{ secrets.READY_URL }}
```

这个示例使用了本仓库内的 composite action，下一节创建。

## 5. 创建 SSH deploy composite action

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

## 6. 注意 sha tag 是否一致

第 4 阶段 `docker/metadata-action` 可能生成短 SHA tag，例如：

```text
sha-8f3a2c1
```

而 `${{ github.sha }}` 是完整 SHA。

你必须让 image workflow 和 deploy workflow 使用一致 tag。本教程中的自动 staging 部署使用短 SHA：

```text
sha-${GITHUB_SHA::7}
```

这和 `docker/metadata-action` 的默认短 SHA 行为更接近。

简单做法：

- 在 image workflow 和 deploy workflow 都使用短 SHA。
- 或在 image workflow 明确生成完整 SHA tag，并同步修改 deploy workflow。
- 或在 deploy workflow 使用同样的 metadata 规则。

初学阶段也可以手动 `workflow_dispatch` 输入具体镜像，确认链路跑通。

## 7. 更简单的第一版

如果 composite action 让你分心，可以先把 SSH steps 直接写在 deploy job 里。

但项目变大后，提取成本地 action 更整洁。

## 8. production 审批

当 job 使用：

```yaml
environment:
  name: production
```

且 GitHub environment 配置了 required reviewers 时，workflow 会等待审批。

审批通过前：

- job 不会继续执行。
- environment secrets 不会暴露给 job。

这正是 production 部署需要的门禁。

## 9. 小练习

1. 创建 `staging` 和 `production` environments。
2. 给 production 添加 required reviewer。
3. 配置 environment secrets。
4. 创建 `.github/workflows/deploy.yml`。
5. 创建 `.github/actions/ssh-deploy/action.yml`。
6. 先用 `workflow_dispatch` 部署 staging。
7. 再用 tag 触发 production，观察审批。

## 10. 本节小结

你现在应该理解：

- GitHub environments 可以管理环境密钥和审批。
- main 可以自动部署 staging。
- tag 可以触发 production 部署。
- production job 应该等待 required reviewers。
- deploy workflow 要和 image workflow 的 tag 策略保持一致。
