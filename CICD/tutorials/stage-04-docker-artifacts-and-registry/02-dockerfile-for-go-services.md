# 02：Go 服务 Dockerfile

## 1. 本节目标

这一节学习如何为 Go 后端服务编写 Dockerfile。

目标：

- 使用多阶段构建。
- 构建 Linux 二进制。
- 使用轻量 runtime 镜像。
- 使用非 root 用户运行。
- 支持版本信息注入。

## 2. 什么是多阶段构建

多阶段构建把“编译环境”和“运行环境”分开。

```text
builder stage:
  - 使用 golang 镜像
  - 下载依赖
  - 编译 Go 二进制

runtime stage:
  - 使用更小的运行时镜像
  - 只复制编译好的二进制
  - 启动服务
```

好处：

- 最终镜像更小。
- 不包含 Go 编译器。
- 不包含源码和构建缓存。
- 攻击面更小。

## 3. 推荐 Dockerfile

项目根目录创建：

```text
Dockerfile
```

内容：

```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.25 AS builder

WORKDIR /src

COPY go.mod go.sum ./
RUN go mod download

COPY . .

ARG VERSION=dev
ARG COMMIT=none
ARG DATE=unknown

RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
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

注意：把 `github.com/your-name/go-cicd-lab` 替换成你 `go.mod` 中的真实 module path。

## 4. 为什么先复制 go.mod 和 go.sum

```dockerfile
COPY go.mod go.sum ./
RUN go mod download

COPY . .
```

这样做是为了利用 Docker layer cache。

当业务代码变化但依赖没有变化时，`go mod download` 这一层可以复用缓存。

如果直接：

```dockerfile
COPY . .
RUN go mod download
```

每次代码变化都可能导致依赖下载层失效。

## 5. 为什么使用 CGO_ENABLED=0

```dockerfile
CGO_ENABLED=0
```

表示构建静态链接 Go 二进制，尽量减少运行时系统库依赖。

适合大多数普通 HTTP API 服务。

如果你的项目使用：

- SQLite。
- 需要 CGO 的数据库驱动。
- C/C++ 库。
- 某些系统调用依赖。

那就不能简单照抄，需要改用包含必要系统库的 runtime 镜像。

## 6. 为什么使用 distroless

```dockerfile
FROM gcr.io/distroless/static-debian12:nonroot
```

distroless 镜像只包含运行应用所需的最小文件，不包含 shell、包管理器等工具。

优点：

- 体积小。
- 攻击面小。
- 适合生产运行。

缺点：

- 容器里没有 shell。
- 不能 `docker exec -it container sh` 进去排查。
- 调试更依赖日志和本地 debug 镜像。

初学也可以先用：

```dockerfile
FROM alpine:3.20
```

但生产更推荐逐步习惯 distroless 或类似精简运行时镜像。

## 7. 为什么使用非 root 用户

```dockerfile
USER nonroot:nonroot
```

容器内进程如果以 root 运行，容器逃逸或挂载配置错误时风险更大。

非 root 运行是基础加固措施。

## 8. 版本信息注入

Dockerfile 中：

```dockerfile
ARG VERSION=dev
ARG COMMIT=none
ARG DATE=unknown
```

构建时：

```bash
docker build \
  --build-arg VERSION=v0.1.0 \
  --build-arg COMMIT=$(git rev-parse --short HEAD) \
  --build-arg DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  -t go-cicd-lab:v0.1.0 .
```

Windows PowerShell：

```powershell
$commit = git rev-parse --short HEAD
$date = Get-Date -AsUTC -Format "yyyy-MM-ddTHH:mm:ssZ"
docker build --build-arg VERSION=v0.1.0 --build-arg COMMIT=$commit --build-arg DATE=$date -t go-cicd-lab:v0.1.0 .
```

## 9. 不要在 Dockerfile 中放密钥

不要写：

```dockerfile
ENV DATABASE_URL=postgres://user:password@host/db
```

也不要：

```dockerfile
COPY .env .env
```

密钥应该在运行时通过环境变量、secret manager 或部署平台注入。

## 10. 小练习

1. 为 `go-cicd-lab` 创建 Dockerfile。
2. 替换真实 module path。
3. 本地执行：

```bash
docker build -t go-cicd-lab:local .
```

4. 查看镜像：

```bash
docker images go-cicd-lab
```

## 11. 本节小结

你现在应该理解：

- Go 服务推荐使用多阶段 Dockerfile。
- builder 阶段负责编译，runtime 阶段只运行二进制。
- 先复制 `go.mod` 和 `go.sum` 可以提升缓存命中。
- 非 root 用户和精简 runtime 是基础安全实践。
- 密钥不能进入 Dockerfile 和镜像层。

