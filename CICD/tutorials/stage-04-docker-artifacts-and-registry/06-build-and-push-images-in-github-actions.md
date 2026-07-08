# 06：在 GitHub Actions 中构建并推送镜像

## 1. 本节目标

这一节把本地镜像构建搬进 GitHub Actions。

目标：

```text
PR: 构建镜像但不推送。
push main: 构建并推送 main/sha tag。
push tag v*: 构建并推送 release tag。
```

## 2. 推荐 workflow 结构

建议单独创建：

```text
.github/workflows/image.yml
```

不要一开始把所有内容塞进 `ci.yml`。

这样更清晰：

```text
ci.yml: 代码质量门禁。
image.yml: 镜像构建和发布。
```

## 3. 完整示例：推送到 GHCR

```yaml
name: image

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
    tags:
      - "v*"
  workflow_dispatch:

permissions:
  contents: read
  packages: write

env:
  REGISTRY: ghcr.io

jobs:
  build:
    name: build
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v6

      - name: Set lowercase image name
        run: echo "IMAGE_NAME=${GITHUB_REPOSITORY,,}" >> "$GITHUB_ENV"

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to registry
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix=sha-
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          build-args: |
            VERSION=${{ steps.meta.outputs.version }}
            COMMIT=${{ github.sha }}
            DATE=${{ github.event.repository.updated_at }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## 4. 触发规则解释

```yaml
pull_request:
  branches: [main]
```

PR 时构建镜像，验证 Dockerfile 是否可用，但不推送。

```yaml
push:
  branches: [main]
  tags:
    - "v*"
```

main 分支和版本 tag 推送时构建并推送镜像。

## 5. 为什么 PR 不推送

```yaml
push: ${{ github.event_name != 'pull_request' }}
```

PR 可能来自不受信任代码。

PR 阶段只验证镜像能构建，不应该默认发布镜像到 registry。

## 6. packages: write

```yaml
permissions:
  contents: read
  packages: write
```

推送 GHCR 需要 packages 写权限。

如果只是 PR 构建，不需要 `packages: write`。但同一个 workflow 还负责 main/tag 推送，所以这里给了 packages 写权限。

更严格的做法是拆成两个 workflow 或用 job 级权限。

## 7. metadata-action 做什么

`docker/metadata-action` 会根据 Git 事件生成：

- tags。
- labels。
- version。

例如：

```text
main -> ghcr.io/owner/repo:main
commit -> ghcr.io/owner/repo:sha-8f3a2c1
tag v0.1.0 -> ghcr.io/owner/repo:0.1.0
```

你也可以调整规则，让 tag 保留 `v` 前缀。团队保持一致即可。

## 8. build-push-action 做什么

`docker/build-push-action` 使用 BuildKit/Buildx 构建镜像。

关键配置：

```yaml
context: .
file: ./Dockerfile
push: true/false
tags: ...
labels: ...
cache-from: type=gha
cache-to: type=gha,mode=max
```

第 8 篇会深入 Buildx 和缓存。

## 9. 和 ci.yml 的关系

推荐让 `image.yml` 也先跑基础质量门禁，或者依赖分支保护确保 main 只进入已通过 CI 的代码。

简单学习项目可以：

```text
PR:
  ci.yml 必须通过。
  image.yml 验证 Dockerfile 构建。

main:
  ci.yml 已在 PR 阶段通过。
  image.yml 构建并推送镜像。
```

更严格可以让 image workflow 中也执行 `make ci-core`。

## 10. 小练习

完成：

1. 创建 `.github/workflows/image.yml`。
2. 提交 PR。
3. 观察 PR 中 image workflow 是否只 build 不 push。
4. 合并 main 后查看 GHCR 是否出现镜像。
5. 创建 tag：

```bash
git tag -a v0.1.0 -m "release v0.1.0"
git push origin v0.1.0
```

6. 查看 release tag 镜像是否出现。

## 11. 本节小结

你现在应该理解：

- CI 可以自动构建和推送镜像。
- PR 构建镜像但不推送更安全。
- main/tag 推送镜像到 registry。
- `docker/metadata-action` 管理 tags 和 labels。
- `docker/build-push-action` 负责 Buildx 构建和推送。

