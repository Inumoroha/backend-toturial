# 07：容器镜像安全

## 1. 本节目标

Go 后端最终经常以容器镜像发布。

镜像如果过大、权限过高、带有 secret、依赖存在高危漏洞，就会把风险带到每个环境。

这一节学习：

- 如何编写更安全的 Dockerfile。
- 如何理解基础镜像选择。
- 如何使用非 root 用户。
- 如何扫描镜像漏洞。
- 如何在 CI 中设置镜像安全门禁。

## 2. 镜像安全看什么

一个镜像至少要关注：

```text
基础镜像来源
基础镜像版本
系统包漏洞
应用二进制漏洞
镜像层中是否包含 secret
运行用户是否为 root
镜像入口命令是否明确
是否固定 digest
是否生成 SBOM 和 provenance
```

本节先处理前半部分，SBOM、provenance 和签名会在后面继续学习。

## 3. Go 服务 Dockerfile 基线

示例：

```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.23-alpine AS build
WORKDIR /src

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath -ldflags="-s -w" -o /out/app ./cmd/api

FROM gcr.io/distroless/static-debian12:nonroot
WORKDIR /app
COPY --from=build /out/app /app/app

USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/app/app"]
```

这个例子体现：

- 多阶段构建。
- 运行镜像不包含 Go 编译器。
- 非 root 运行。
- 使用 distroless nonroot 镜像。
- 明确入口命令。

## 4. 基础镜像选择

常见选择：

```text
scratch
distroless
alpine
debian/ubuntu slim
语言官方镜像
```

简单对比：

| 基础镜像 | 优点 | 风险或代价 |
| --- | --- | --- |
| `scratch` | 极小，攻击面低 | 无 shell、无 CA 时要额外处理，排障不方便 |
| `distroless` | 小，适合生产运行 | 排障工具少 |
| `alpine` | 小，有包管理器 | musl 兼容性和系统包漏洞仍需关注 |
| `slim` | 兼容性好 | 体积更大，系统包更多 |
| `golang` | 构建方便 | 不适合作为生产运行镜像 |

Go 静态二进制通常适合：

```text
构建阶段使用 golang 镜像。
运行阶段使用 distroless 或 scratch。
```

## 5. 不要把 secret 放进镜像

危险写法：

```dockerfile
ARG GITHUB_TOKEN
RUN git clone https://${GITHUB_TOKEN}@github.com/your-org/private-module
```

风险：

- token 可能出现在构建日志。
- token 可能进入镜像层历史。
- token 可能被缓存系统保留。

更好的方向：

- 在 CI job 中用最小权限凭证下载依赖。
- 使用 BuildKit secret mount。
- 不把 secret 写入文件系统层。

BuildKit 示例：

```dockerfile
# syntax=docker/dockerfile:1.7

RUN --mount=type=secret,id=git_token \
    TOKEN="$(cat /run/secrets/git_token)" && \
    git config --global url."https://${TOKEN}@github.com/".insteadOf "https://github.com/" && \
    go mod download && \
    git config --global --unset-all url."https://${TOKEN}@github.com/".insteadOf
```

CI 构建时传入：

```bash
docker build \
  --secret id=git_token,env=GIT_TOKEN \
  -t go-cicd-lab:dev .
```

学习阶段如果不需要私有依赖，先不要引入这个复杂度。

## 6. 固定基础镜像 digest

普通写法：

```dockerfile
FROM alpine:3.20
```

更可控：

```dockerfile
FROM alpine:3.20@sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

好处：

- 构建输入更稳定。
- 容易复现。
- 避免 tag 被更新后构建内容悄悄变化。

代价：

- digest 更新需要维护。
- 安全补丁不会自动进入，需要 Dependabot 或人工升级。

建议：

```text
学习阶段使用清晰 tag。
生产关键镜像逐步固定 digest，并建立更新流程。
```

## 7. 本地使用 Trivy 扫描镜像

安装 Trivy 后执行：

```bash
trivy image go-cicd-lab:dev
```

只看高危和严重漏洞：

```bash
trivy image --severity HIGH,CRITICAL go-cicd-lab:dev
```

设置退出码，让命令失败：

```bash
trivy image \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  go-cicd-lab:dev
```

扫描 Dockerfile/IaC 配置：

```bash
trivy config .
```

扫描文件系统：

```bash
trivy fs .
```

## 8. CI 中扫描镜像

示例：

```yaml
name: image-security

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t go-cicd-lab:${{ github.sha }} .

      - name: Scan image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: go-cicd-lab:${{ github.sha }}
          format: table
          severity: HIGH,CRITICAL
          exit-code: "1"
```

生产中请按官方文档确认 action 的最新推荐版本，关键 workflow 建议 pin 到 commit SHA。

如果要上传 SARIF 到 GitHub code scanning：

需要增加权限：

```yaml
permissions:
  contents: read
  security-events: write
```

```yaml
      - name: Trivy SARIF
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: go-cicd-lab:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif
```

## 9. 漏洞门禁策略

不要一开始就追求“所有漏洞 0 个”，否则团队可能会绕过扫描。

更实用的策略：

```text
CRITICAL 阻断。
HIGH 阻断或要求安全负责人审批。
MEDIUM 建 issue 并限期修复。
LOW 记录即可。
没有修复版本时记录风险接受。
```

注意：

- 扫描结果会随数据库更新变化。
- 老镜像可能今天通过，明天失败。
- 这不是坏事，它说明你发现了新的风险。

## 10. 运行时安全补充

镜像本身之外，Kubernetes 运行时还应考虑：

```yaml
securityContext:
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

不是所有应用都能立刻开启只读根文件系统。

如果应用要写临时文件，可以挂载：

```yaml
volumes:
  - name: tmp
    emptyDir: {}

volumeMounts:
  - name: tmp
    mountPath: /tmp
```

## 11. 小练习

对你的 Go 服务完成：

1. 确认 Dockerfile 是多阶段构建。
2. 确认生产运行镜像不是 `golang`。
3. 增加 `USER nonroot:nonroot` 或等价配置。
4. 本地运行 `trivy image`。
5. 在 CI 中加入镜像扫描。
6. 制定 `CRITICAL` 漏洞阻断策略。

## 12. 本节小结

你现在应该理解：

- 镜像安全从 Dockerfile 和基础镜像开始。
- 运行镜像应尽量小，不带编译器和构建工具。
- 不要把 secret 写进镜像层。
- 高危镜像漏洞应在 CI 阶段暴露出来。
- digest pinning 提高可复现性，但需要更新机制配合。
