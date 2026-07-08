# 10：复习清单与自测题

## 1. 本节目标

用清单和问题确认你是否掌握第 4 阶段。

如果你能独立为 Go 服务构建、标记、扫描、推送镜像，就可以进入第 5 阶段：CD 基础。

## 2. 操作清单

### Dockerfile

- [ ] 我能解释多阶段构建。
- [ ] 我能写 Go 服务 Dockerfile。
- [ ] 我知道为什么先复制 `go.mod` 和 `go.sum`。
- [ ] 我知道 `CGO_ENABLED=0` 的作用和限制。
- [ ] 我知道为什么最终镜像不应该包含源码和编译器。
- [ ] 我知道为什么要使用非 root 用户。

### 本地镜像操作

- [ ] 我能编写 `.dockerignore`。
- [ ] 我能执行 `docker build`。
- [ ] 我能执行 `docker run`。
- [ ] 我能查看 `docker logs`。
- [ ] 我能执行 `docker inspect`。
- [ ] 我能清理本地镜像。

### 镜像版本

- [ ] 我知道 tag 和 digest 的区别。
- [ ] 我知道为什么生产不应该只依赖 `latest`。
- [ ] 我能设计 main、sha、release tag。
- [ ] 我知道 OCI labels 的作用。
- [ ] 我能把镜像追溯到 commit。

### 镜像仓库

- [ ] 我能登录 registry。
- [ ] 我能给镜像重新 tag。
- [ ] 我能 push 镜像。
- [ ] 我能 pull 镜像。
- [ ] 我知道 GHCR、GitLab Registry、Docker Hub 的基本差异。
- [ ] 我知道镜像名通常要小写。

### GitHub Actions

- [ ] 我能创建 `.github/workflows/image.yml`。
- [ ] 我能使用 `docker/setup-buildx-action`。
- [ ] 我能使用 `docker/login-action`。
- [ ] 我能使用 `docker/metadata-action`。
- [ ] 我能使用 `docker/build-push-action`。
- [ ] 我知道 PR 构建不应该默认 push。
- [ ] 我知道推送 GHCR 需要 `packages: write`。

### 扫描与缓存

- [ ] 我能用 Trivy 扫描镜像。
- [ ] 我知道 HIGH/CRITICAL 漏洞通常要阻断。
- [ ] 我知道 SBOM 是什么。
- [ ] 我能生成 SBOM。
- [ ] 我知道 Buildx cache 是优化，不是正确性依赖。
- [ ] 我知道多架构镜像适合什么场景。

## 3. 自测题

### 题目 1：为什么 Go 服务推荐多阶段 Dockerfile？

参考答案：

```text
builder 阶段包含 Go 编译器和源码，用于构建二进制。
runtime 阶段只复制编译好的二进制和必要运行时文件。
这样最终镜像更小、攻击面更低，也不包含源码和构建缓存。
```

### 题目 2：`.dockerignore` 解决什么问题？

参考答案：

```text
它减少 Docker build context，避免把 .git、bin、coverage、.env、私钥等不需要或敏感文件发送进构建上下文。
这能提升构建速度、提高缓存稳定性、降低泄密风险。
```

### 题目 3：tag 和 digest 有什么区别？

参考答案：

```text
tag 是镜像的人类可读名称，可能被移动。
digest 是镜像内容的不可变哈希，能精确标识镜像内容。
生产严格部署更适合使用 digest 或至少使用不可变版本 tag。
```

### 题目 4：为什么不要把 `.env` COPY 进镜像？

参考答案：

```text
.env 往往包含数据库密码、API token 等密钥。
镜像会被缓存、推送、下载和扫描，密钥一旦进入镜像层很难彻底清理。
密钥应该在运行时由部署环境注入。
```

### 题目 5：PR 中为什么通常只 build 镜像，不 push？

参考答案：

```text
PR 代码可能来自不受信任来源。
PR 阶段验证 Dockerfile 和镜像构建是否可用即可，不应该默认发布镜像或使用 registry 写权限。
```

### 题目 6：`packages: write` 是做什么的？

参考答案：

```text
在 GitHub Actions 中推送镜像到 GitHub Container Registry 需要写 package 权限。
普通 CI 检查不需要这个权限，只有镜像发布 job 才需要。
```

### 题目 7：SBOM 有什么价值？

参考答案：

```text
SBOM 记录软件或镜像包含的组件和依赖。
它有助于漏洞响应、依赖审计、供应链安全和合规要求。
```

### 题目 8：什么时候需要多架构镜像？

参考答案：

```text
当同一个镜像 tag 需要同时运行在 linux/amd64 和 linux/arm64 等不同架构机器上时，需要多架构镜像。
普通单一部署环境可以先只构建一个架构。
```

## 4. 情景题

### 情景 1

CI 中 Docker build 失败，日志显示找不到 `go.sum`。

你会怎么排查？

参考思路：

```text
确认项目是否有 go.sum。
如果没有，运行 go mod tidy。
确认 go.sum 已提交。
检查 .dockerignore 是否误排除了 go.sum。
检查 Dockerfile 中 COPY 路径是否正确。
```

### 情景 2

镜像推送到 GHCR 失败，提示权限不足。

你会检查什么？

参考思路：

```text
检查 workflow permissions 是否包含 packages: write。
检查是否在 pull_request 事件中尝试 push。
检查 docker/login-action 的 registry、username、password。
检查 package 权限和仓库可见性。
```

### 情景 3

生产部署使用 `latest`，现在出现问题，但没人知道上一个稳定版本是什么。

你会如何改进？

参考思路：

```text
停止把 latest 作为生产唯一依据。
为每次构建生成 sha tag，为正式发布生成语义化版本 tag。
记录生产部署的 tag 和 digest。
保留最近稳定镜像，用明确版本回滚。
```

## 5. 第 4 阶段最终作业

在 `go-cicd-lab` 中完成：

```text
Dockerfile
.dockerignore
.github/workflows/image.yml
docs/container-image.md
```

要求：

- 使用多阶段构建。
- 最终镜像使用非 root 用户。
- `.dockerignore` 排除 `.env`、`.git`、`bin` 等。
- 本地 `docker build` 成功。
- 本地 `docker run` 成功。
- PR 只构建不推送。
- main 推送镜像到 registry。
- tag `v*` 推送 release 镜像。
- 镜像包含版本和 commit 信息。
- 至少能用 Trivy 本地扫描一次镜像。

## 6. 进入下一阶段的标准

当你能做到下面几件事，就可以进入第 5 阶段：

1. 独立写出 Go 服务 Dockerfile。
2. 能本地构建和运行镜像。
3. 能解释镜像 tag、label、digest。
4. 能把镜像推送到 registry。
5. 能在 GitHub Actions 中自动构建和推送镜像。
6. 能说明镜像扫描和 SBOM 的意义。

第 5 阶段会学习如何把镜像部署到环境中，包括 Docker Compose/VM 部署、staging、production、健康检查和回滚。

