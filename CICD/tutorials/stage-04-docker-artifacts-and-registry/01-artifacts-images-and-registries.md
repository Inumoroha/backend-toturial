# 01：制品、镜像与镜像仓库

## 1. 本节目标

第 3 阶段的 CI 回答了：

```text
这次代码能不能合并？
```

第 4 阶段开始回答：

```text
这次代码能不能变成可发布制品？
```

Go 后端项目最常见的发布制品就是容器镜像。

## 2. 什么是制品

制品是构建过程产生的、可以保存、传递、部署或审计的结果。

Go 项目常见制品：

- Go 二进制。
- Docker/OCI 镜像。
- 测试报告。
- 覆盖率报告。
- SBOM。
- Helm Chart。

第 4 阶段重点是：

```text
Docker/OCI image
```

## 3. 为什么后端服务常用镜像发布

容器镜像把应用和运行时依赖打包在一起。

它解决的问题：

- 部署环境更一致。
- 发布制品可版本化。
- 可以推送到镜像仓库。
- 可以被 Kubernetes、Docker Compose、云平台拉取运行。
- 可以扫描漏洞。
- 可以通过 digest 精确追踪。

这比“在开发者电脑上编译一个二进制，然后手动复制到服务器”可靠得多。

## 4. 镜像不是虚拟机

容器镜像不是完整虚拟机。

它通常包含：

- 应用二进制。
- 必要运行时文件。
- CA 证书。
- 系统库，取决于基础镜像。
- 默认启动命令。

它不应该包含：

- 源码，除非确实需要。
- Git 历史。
- 本地缓存。
- `.env`。
- 私钥。
- 生产数据库密码。
- 开发者本机配置。

## 5. 镜像仓库是什么

镜像仓库用于保存和分发镜像。

常见仓库：

- GitHub Container Registry：`ghcr.io`
- GitLab Container Registry：`registry.gitlab.com`
- Docker Hub：`docker.io`
- 云厂商镜像仓库：ECR、GCR/Artifact Registry、ACR 等。

镜像完整名称通常长这样：

```text
ghcr.io/your-name/go-cicd-lab:v0.1.0
```

拆开：

```text
registry: ghcr.io
namespace: your-name
image name: go-cicd-lab
tag: v0.1.0
```

## 6. 第 4 阶段的交付链路

本阶段目标链路：

```text
push main or tag
-> run CI quality gates
-> build Docker image
-> tag image
-> scan image
-> push image to registry
-> save metadata
```

注意：还不部署。

第 5 阶段才会学习：

```text
pull image -> run/deploy -> health check -> rollback
```

## 7. 镜像质量标准

一个好的后端服务镜像应该：

- 能本地运行。
- 体积合理。
- 不包含密钥。
- 使用明确 tag。
- 可以追溯到 commit。
- 可以被扫描。
- 以非 root 用户运行。
- 启动命令清晰。
- 运行时只包含必要文件。

## 8. 镜像和 CI 的关系

CI 负责自动构建镜像。

但构建镜像前，应该先通过质量门禁：

```text
fmt/lint/test/race/vuln/build passed
-> build image
```

不要让明显失败的代码进入镜像仓库。

## 9. 小练习

请为你的项目写出：

| 问题 | 你的答案 |
| --- | --- |
| 镜像仓库选哪个？ | |
| 镜像名称是什么？ | |
| main 分支镜像 tag 怎么命名？ | |
| release tag 镜像怎么命名？ | |
| 镜像是否需要公开？ | |
| 谁有 push 权限？ | |
| 失败镜像是否保留？ | |

## 10. 本节小结

你现在应该理解：

- 第 4 阶段关注可发布制品，不关注部署。
- Go 后端常见发布制品是容器镜像。
- 镜像仓库用于保存和分发镜像。
- 好镜像应该可运行、可追溯、可扫描、无密钥。

