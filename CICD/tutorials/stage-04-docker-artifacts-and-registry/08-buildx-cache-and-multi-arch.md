# 08：Buildx、缓存与多架构镜像

## 1. 本节目标

这一节学习更工程化的镜像构建能力：

- Buildx。
- BuildKit cache。
- GitHub Actions cache。
- 多架构镜像。

先理解概念，再决定是否在项目里启用。

## 2. Buildx 是什么

Docker Buildx 是 Docker 的扩展构建工具，基于 BuildKit。

它支持：

- 更强的缓存。
- 多平台构建。
- 构建输出控制。
- GitHub Actions 集成。

查看版本：

```bash
docker buildx version
```

创建 builder：

```bash
docker buildx create --use
```

查看 builder：

```bash
docker buildx ls
```

## 3. 本地 Buildx 构建

普通构建：

```bash
docker buildx build -t go-cicd-lab:local --load .
```

`--load` 表示把构建结果加载到本地 Docker image store，方便 `docker run`。

如果是推送到 registry：

```bash
docker buildx build -t ghcr.io/your-name/go-cicd-lab:v0.1.0 --push .
```

## 4. 多架构是什么

常见平台：

```text
linux/amd64
linux/arm64
```

例如：

- 大多数传统服务器：`linux/amd64`。
- Apple Silicon、本地 ARM 机器、部分云服务器：`linux/arm64`。

多架构镜像让同一个 tag 可以在不同 CPU 架构上拉取对应镜像。

## 5. Go Dockerfile 支持多架构

修改 builder stage：

```dockerfile
# syntax=docker/dockerfile:1

FROM --platform=$BUILDPLATFORM golang:1.25 AS builder

WORKDIR /src

COPY go.mod go.sum ./
RUN go mod download
COPY . .

ARG TARGETOS
ARG TARGETARCH
ARG VERSION=dev
ARG COMMIT=none
ARG DATE=unknown

RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH go build \
    -trimpath \
    -ldflags="-s -w" \
    -o /out/server ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /out/server /server
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/server"]
```

Buildx 会传入 `TARGETOS` 和 `TARGETARCH`。

## 6. 本地构建多架构并推送

多架构通常需要直接 push：

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/your-name/go-cicd-lab:v0.1.0 \
  --push .
```

如果只是本地测试单架构：

```bash
docker buildx build --platform linux/amd64 -t go-cicd-lab:local --load .
```

## 7. GitHub Actions 中的 Buildx 缓存

```yaml
- name: Build and push
  uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: ${{ steps.meta.outputs.tags }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

`type=gha` 使用 GitHub Actions cache backend。

好处：

- 依赖层和构建层可以复用。
- main/tag 构建更快。

注意：

- 缓存是优化，不是正确性依赖。
- 缓存命中失败时构建也应该成功，只是慢。

## 8. 多架构 CI 示例

```yaml
- name: Build and push
  uses: docker/build-push-action@v6
  with:
    context: .
    platforms: linux/amd64,linux/arm64
    push: ${{ github.event_name != 'pull_request' }}
    tags: ${{ steps.meta.outputs.tags }}
    labels: ${{ steps.meta.outputs.labels }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

初学项目可以先单架构。

等你需要支持 ARM 机器或 Apple Silicon 本地运行，再开启多架构。

## 9. 多架构的常见坑

- CGO 项目交叉编译更复杂。
- 某些基础镜像不支持所有平台。
- 构建时间变长。
- 本地 `--load` 不支持加载完整多架构 manifest。
- 某些依赖在 ARM 上行为不同。

Go 纯静态服务通常比较适合多架构。

## 10. 小练习

1. 查看本机架构：

```bash
docker version
```

2. 用 Buildx 构建本地镜像：

```bash
docker buildx build -t go-cicd-lab:buildx --load .
```

3. 如果你有 registry，尝试多架构推送：

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t <registry>/<namespace>/go-cicd-lab:multiarch-test \
  --push .
```

## 11. 本节小结

你现在应该理解：

- Buildx 是更强的 Docker 构建工具。
- `cache-from` 和 `cache-to` 可以加速 CI 构建。
- 多架构镜像适合需要同时支持 amd64 和 arm64 的场景。
- 多架构不是第一天必须做，先保证单架构稳定。

