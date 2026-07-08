# 06：Docker 构建缓存与镜像优化

## 1. 本节目标

Go 服务的镜像构建如果设计不好，每次 CI 都会重复：

```text
下载 Go module
重新编译所有包
复制无关文件
重新扫描大镜像
```

这一节学习如何优化 Dockerfile 和 buildx cache。

## 2. Docker 缓存的基本原则

Docker 按层缓存。

只要某一层输入变化，它之后的层通常都要重新执行。

错误顺序：

```dockerfile
COPY . .
RUN go mod download
RUN go build -o app ./cmd/api
```

问题：

```text
任何源码变化都会导致 go mod download 重新执行。
```

更好：

```dockerfile
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN go build -o app ./cmd/api
```

依赖文件不变时，`go mod download` 层可以复用。

## 3. 准备 .dockerignore

创建：

```text
.dockerignore
```

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
node_modules
```

目的：

- 减少 build context。
- 避免把无关文件传给 Docker daemon。
- 降低 secret 误入构建上下文的风险。

注意：

```text
不要忽略构建必需文件，例如 go.mod、go.sum、cmd、internal、pkg。
```

## 4. 使用 BuildKit cache mount

Dockerfile 示例：

```dockerfile
# syntax=docker/dockerfile:1.7

FROM golang:1.23-alpine AS build
WORKDIR /src

COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -trimpath -o /out/app ./cmd/api

FROM gcr.io/distroless/static-debian12:nonroot
WORKDIR /app
COPY --from=build /out/app /app/app
USER nonroot:nonroot
ENTRYPOINT ["/app/app"]
```

这能让 Go module 和 build cache 在 BuildKit 构建中复用。

## 5. 使用 buildx 的 GitHub Actions cache

示例 workflow：

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build image
  uses: docker/build-push-action@v6
  with:
    context: .
    push: false
    tags: go-cicd-lab:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

推送镜像时：

```yaml
- name: Build and push
  uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

说明：

- `cache-from` 读取之前缓存。
- `cache-to` 写入新缓存。
- `mode=max` 缓存更多层，空间占用也更大。

## 6. 多平台构建的成本

多平台构建：

```yaml
platforms: linux/amd64,linux/arm64
```

可能显著变慢，尤其在模拟架构时。

建议：

```text
PR：只构建 linux/amd64 或只做 Dockerfile 校验。
main：构建生产需要的平台。
release：构建完整多平台镜像。
```

不要每个 PR 都做完整 release 级构建，除非确实有必要。

## 7. 镜像大小优化

查看镜像大小：

```bash
docker images go-cicd-lab
```

查看层：

```bash
docker history go-cicd-lab:dev
```

优化方向：

- 多阶段构建。
- 使用 distroless/scratch 运行镜像。
- 不复制源码到运行镜像。
- 不安装调试工具到生产镜像。
- `go build -ldflags="-s -w"` 减小二进制。

不要为了极限小牺牲可维护性。

例如 `scratch` 镜像需要处理 CA 证书、时区、用户等问题。

## 8. 扫描速度优化

镜像越大，扫描通常越慢。

但不要通过跳过扫描来优化。

可以做：

- 减小运行镜像。
- 避免无关系统包。
- PR 阶段扫描本地镜像。
- main/release 阶段扫描推送后的 digest。
- 缓存扫描数据库时按工具官方建议配置。

## 9. 验证优化效果

记录：

```text
build context 大小
Docker build 耗时
cache hit/miss
镜像大小
Trivy scan 耗时
```

示例表：

```markdown
| Item | Before | After |
| --- | --- | --- |
| Build duration | 8m20s | 3m10s |
| Image size | 92MB | 18MB |
| Scan duration | 2m00s | 45s |
```

## 10. 小练习

完成：

1. 为项目增加 `.dockerignore`。
2. 调整 Dockerfile，使 `go mod download` 在复制源码前执行。
3. 使用 BuildKit cache mount。
4. 在 GitHub Actions 中加入 `cache-from: type=gha` 和 `cache-to: type=gha,mode=max`。
5. 比较优化前后镜像构建耗时和大小。

## 11. 本节小结

你现在应该理解：

- Dockerfile 层顺序决定缓存效果。
- `.dockerignore` 会影响构建速度和安全性。
- BuildKit cache mount 能加速 Go 构建。
- buildx 的 `type=gha` 可以在 GitHub Actions 中复用构建缓存。
- 多平台构建和镜像扫描要按 PR/main/release 分层执行。

