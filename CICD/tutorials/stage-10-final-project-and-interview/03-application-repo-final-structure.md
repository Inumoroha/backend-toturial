# 03：应用仓库最终结构

## 1. 本节目标

应用仓库是毕业项目的主体。

它要展示：

```text
Go 工程化能力
本地可运行能力
CI 质量门禁
镜像构建和发布能力
安全与可观测能力
```

这一节整理 `go-cicd-lab` 的最终结构。

## 2. 推荐目录

```text
go-cicd-lab/
  cmd/
    api/
      main.go
  internal/
    config/
    httpapi/
    repository/
    service/
  migrations/
  scripts/
    smoke-test.sh
  docs/
    architecture.md
    ci-cd.md
    security.md
    security-incident-runbook.md
    cicd-observability.md
    pipeline-optimization-report.md
    release-checklist.md
  .github/
    workflows/
      ci.yml
      codeql.yml
      image.yml
      update-deploy-repo.yml
    dependabot.yml
    pull_request_template.md
  Dockerfile
  .dockerignore
  Makefile
  go.mod
  go.sum
  README.md
```

不必一开始完全相同，但要能解释每个目录的职责。

## 3. Makefile 最终形态

示例：

```makefile
.PHONY: fmt test test-race lint vuln build docker-build smoke

APP_NAME := go-cicd-lab
IMAGE ?= ghcr.io/your-org/go-cicd-lab:dev

fmt:
	go fmt ./...

test:
	go test ./...

test-race:
	go test -race ./...

lint:
	golangci-lint run

vuln:
	govulncheck ./...

build:
	CGO_ENABLED=0 go build -trimpath -o bin/$(APP_NAME) ./cmd/api

docker-build:
	docker build -t $(IMAGE) .

smoke:
	bash scripts/smoke-test.sh
```

原则：

```text
本地命令和 CI 命令保持一致。
CI 只是调用这些命令，而不是另起一套逻辑。
```

## 4. Dockerfile 最终形态

示例：

```dockerfile
# syntax=docker/dockerfile:1.7

FROM golang:1.23-alpine AS build
WORKDIR /src

COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

COPY . .
ARG VERSION=dev
ARG COMMIT=unknown
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -trimpath \
      -ldflags="-s -w -X main.version=${VERSION} -X main.commit=${COMMIT}" \
      -o /out/app ./cmd/api

FROM gcr.io/distroless/static-debian12:nonroot
WORKDIR /app
COPY --from=build /out/app /app/app
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/app/app"]
```

生产项目中 Go 和基础镜像版本请按团队标准和官方文档确认。

## 5. .dockerignore

示例：

```dockerignore
.git
.github
bin
dist
coverage.out
reports
tmp
*.log
.env
.env.*
docs
```

如果 Docker build 需要文档或脚本，不要忽略它们。

## 6. README 必须回答的问题

应用仓库 README 至少包含：

- 项目简介。
- 架构图或流程图。
- 本地运行方式。
- 本地测试方式。
- Docker 构建方式。
- CI/CD 流程说明。
- 环境变量说明。
- 发布和回滚入口。
- 安全和可观测文档链接。

示例目录：

```markdown
# go-cicd-lab

## Overview

## Architecture

## Local Development

## Testing

## Docker

## CI/CD

## Release

## Observability

## Security

## Troubleshooting
```

## 7. Workflow 文件职责

建议：

| 文件 | 作用 |
| --- | --- |
| `ci.yml` | PR 测试、lint、漏洞检查 |
| `codeql.yml` | 代码扫描 |
| `image.yml` | 构建、扫描、签名、推送镜像 |
| `update-deploy-repo.yml` | 更新部署仓库 staging 或创建 production PR |

不要把所有逻辑塞进一个巨大 workflow。

## 8. 文档不要空转

每个文档都要有用途：

| 文档 | 用途 |
| --- | --- |
| `architecture.md` | 解释项目结构和架构 |
| `ci-cd.md` | 解释流水线设计 |
| `security.md` | 解释安全基线 |
| `release-checklist.md` | 发布前检查 |
| `security-incident-runbook.md` | 安全事件响应 |
| `cicd-observability.md` | 指标和观测策略 |
| `pipeline-optimization-report.md` | 优化前后对比 |

## 9. 小练习

在应用仓库执行：

```bash
mkdir -p docs scripts .github/workflows
```

然后创建：

```text
docs/architecture.md
docs/ci-cd.md
docs/release-checklist.md
scripts/smoke-test.sh
```

最后检查：

```bash
make test
make build
make docker-build
```

## 10. 本节小结

你现在应该有一个清晰的应用仓库蓝图。

毕业项目的应用仓库要做到：

- 本地可运行。
- CI 可复用本地命令。
- 镜像可构建可发布。
- 文档能解释设计。
- 面试时能从目录讲到流水线。

