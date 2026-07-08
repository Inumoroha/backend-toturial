# 09：实战，为 go-cicd-lab 构建和发布镜像

## 1. 本节目标

这一节把第 4 阶段落到 `go-cicd-lab`。

最终产出：

```text
Dockerfile
.dockerignore
.github/workflows/image.yml
docs/container-image.md
```

并实现：

```text
PR: 验证镜像能构建。
main: 构建并推送镜像到 GHCR。
tag v*: 构建并推送 release 镜像。
```

## 2. 前置检查

本地先确认：

```bash
go test ./...
go build ./...
docker version
docker buildx version
```

项目需要有服务入口：

```text
cmd/server/main.go
```

如果服务还没有真正监听 HTTP，也可以先用本阶段练习 Dockerfile。后面第 5 阶段部署时再补完整 HTTP 服务。

## 3. 添加 Dockerfile

创建：

```text
Dockerfile
```

内容：

```dockerfile
# syntax=docker/dockerfile:1

FROM --platform=$BUILDPLATFORM golang:1.25 AS builder

WORKDIR /src

COPY go.mod go.sum ./
RUN go mod download

COPY . .

ARG TARGETOS=linux
ARG TARGETARCH=amd64
ARG VERSION=dev
ARG COMMIT=none
ARG DATE=unknown

RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH go build \
    -trimpath \
    -ldflags="-s -w \
    -X 'github.com/your-name/go-cicd-lab/internal/version.Version=${VERSION}' \
    -X 'github.com/your-name/go-cicd-lab/internal/version.Commit=${COMMIT}' \
    -X 'github.com/your-name/go-cicd-lab/internal/version.Date=${DATE}'" \
    -o /out/server ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot

WORKDIR /
COPY --from=builder /out/server /server

USER nonroot:nonroot
EXPOSE 8080

ENTRYPOINT ["/server"]
```

把 `github.com/your-name/go-cicd-lab` 替换成真实 module path。

## 4. 添加 .dockerignore

创建：

```text
.dockerignore
```

内容：

```dockerignore
.git
.github
bin
coverage.out
*.test
*.out
.env
.env.*
!.env.example
*.pem
*.key
tmp
dist
```

## 5. 本地构建

```bash
docker build -t go-cicd-lab:local .
```

带版本参数：

```bash
docker build \
  --build-arg VERSION=local \
  --build-arg COMMIT=$(git rev-parse --short HEAD) \
  --build-arg DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  -t go-cicd-lab:local .
```

PowerShell：

```powershell
$commit = git rev-parse --short HEAD
$date = Get-Date -AsUTC -Format "yyyy-MM-ddTHH:mm:ssZ"
docker build --build-arg VERSION=local --build-arg COMMIT=$commit --build-arg DATE=$date -t go-cicd-lab:local .
```

## 6. 本地运行

```bash
docker run --rm -p 8080:8080 go-cicd-lab:local
```

如果服务有 `/healthz`：

```bash
curl -f http://localhost:8080/healthz
```

如果当前服务只是打印内容后退出，也没关系。先确认镜像能构建，后续阶段再完善运行时服务。

## 7. 添加 image workflow

创建：

```text
.github/workflows/image.yml
```

内容：

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

      - name: Set build date
        run: echo "BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)" >> "$GITHUB_ENV"

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
            DATE=${{ env.BUILD_DATE }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## 8. 可选：加入 Trivy 扫描

如果你想在同一个 workflow 中扫描，可以先构建本地镜像 tag，再扫描。

更简单的学习路线：

1. 先让 image workflow build/push 变绿。
2. 再单独增加 Trivy job。

示例：

```yaml
  scan:
    name: scan
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name != 'pull_request'
    steps:
      - name: Set lowercase image name
        run: echo "IMAGE_NAME=${GITHUB_REPOSITORY,,}" >> "$GITHUB_ENV"
      - name: Scan image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/${{ env.IMAGE_NAME }}:sha-${{ github.sha }}
          format: table
          exit-code: "1"
          severity: HIGH,CRITICAL
```

注意：`docker/metadata-action` 默认生成的短 SHA tag 和这里的完整 `${{ github.sha }}` 可能不一致。实际使用时要统一 tag 规则。初学阶段可以先手动确认 tag，再加入扫描。

## 9. 添加文档

创建：

```text
docs/container-image.md
```

内容模板：

````markdown
# 容器镜像说明

## 镜像仓库

- Registry：GHCR
- Image：`ghcr.io/<owner>/<repo>`

## 本地构建

```bash
docker build -t go-cicd-lab:local .
```

## 本地运行

```bash
docker run --rm -p 8080:8080 go-cicd-lab:local
```

## Tag 策略

- PR：只构建，不推送。
- main：推送 branch tag 和 sha tag。
- `v*`：推送 release tag。

## 安全规则

- 不把 `.env` 复制进镜像。
- 不把私钥复制进镜像。
- 使用多阶段构建。
- 使用非 root 用户运行。

## 回滚

回滚时选择上一个稳定镜像 tag 或 digest。
````

## 10. 提交 PR

```bash
git switch -c ci/add-container-image-workflow
git add Dockerfile .dockerignore .github/workflows/image.yml docs/container-image.md
git commit -m "ci: add container image build workflow"
git push -u origin ci/add-container-image-workflow
```

创建 PR，观察：

- `ci` workflow 是否通过。
- `image` workflow 是否 build 成功。
- PR 是否没有推送镜像。

合并 main 后，查看 GHCR 是否出现镜像。

## 11. 发布 release 镜像

```bash
git switch main
git pull origin main
git tag -a v0.1.0 -m "release v0.1.0"
git push origin v0.1.0
```

查看 GHCR 是否出现：

```text
v0.1.0
sha-xxxxxxx
```

## 12. 本节验收

你完成后应该能做到：

- 本地 `docker build` 成功。
- 本地 `docker run` 可以启动容器。
- PR 自动验证镜像构建。
- main 自动推送镜像到 GHCR。
- tag `v*` 自动推送 release 镜像。
- 文档说明镜像命名、tag 策略和回滚方式。

## 13. 本节小结

你现在已经把 Go 后端项目推进到了“有可发布镜像”的阶段。

下一阶段会学习如何把这个镜像部署到 staging 或生产环境，并设计健康检查和回滚流程。

