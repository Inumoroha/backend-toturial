# 阶段 4：Docker、镜像与制品管理

> 本阶段目标：把 Go 后端服务从“代码能通过 CI”推进到“能构建出可发布、可追溯、可扫描、可推送的容器镜像”。

## 学习顺序

请按下面顺序学习：

1. [01-artifacts-images-and-registries.md](./01-artifacts-images-and-registries.md)
   - 理解制品、容器镜像、镜像仓库。
   - 明确第 4 阶段只构建和发布镜像，不做部署。

2. [02-dockerfile-for-go-services.md](./02-dockerfile-for-go-services.md)
   - 学习 Go 服务多阶段 Dockerfile。
   - 理解 builder 镜像、runtime 镜像、非 root 用户和静态二进制。

3. [03-dockerignore-build-run-and-debug.md](./03-dockerignore-build-run-and-debug.md)
   - 学习 `.dockerignore`。
   - 掌握本地 `docker build`、`docker run`、`docker logs`、`docker inspect`。

4. [04-image-tags-labels-and-versioning.md](./04-image-tags-labels-and-versioning.md)
   - 学习镜像 tag、label、digest。
   - 设计 Git SHA、分支、语义化版本的标签策略。

5. [05-container-registries-ghcr-gitlab-dockerhub.md](./05-container-registries-ghcr-gitlab-dockerhub.md)
   - 学习 GHCR、GitLab Container Registry、Docker Hub。
   - 掌握 login、tag、push、pull 的基本命令。

6. [06-build-and-push-images-in-github-actions.md](./06-build-and-push-images-in-github-actions.md)
   - 在 GitHub Actions 中构建并推送镜像。
   - 使用 Docker 官方 Actions 生成 metadata、登录 registry、build/push。

7. [07-image-scanning-sbom-and-basic-hardening.md](./07-image-scanning-sbom-and-basic-hardening.md)
   - 学习镜像扫描、SBOM 和基础加固。
   - 用 Trivy 扫描镜像并在 CI 中阻断高危漏洞。

8. [08-buildx-cache-and-multi-arch.md](./08-buildx-cache-and-multi-arch.md)
   - 学习 Docker Buildx、构建缓存、多架构镜像。
   - 理解 `linux/amd64` 和 `linux/arm64`。

9. [09-practice-containerize-go-cicd-lab.md](./09-practice-containerize-go-cicd-lab.md)
   - 为 `go-cicd-lab` 落地 Dockerfile、`.dockerignore`、镜像发布 workflow。
   - 输出 `docs/container-image.md`。

10. [10-review-checklist-and-quiz.md](./10-review-checklist-and-quiz.md)
    - 用清单和自测题检查是否可以进入第 5 阶段。

## 本阶段建议时间

- 快速学习：3 到 5 天。
- 扎实学习：1 到 2 周。
- 推荐方式：先本地构建运行镜像，再把构建推送放进 CI。

## 本阶段你要准备什么

- 已完成第 3 阶段的 Go CI。
- 本地安装 Docker Desktop 或 Docker Engine。
- 一个镜像仓库账号，推荐先用 GitHub Container Registry。
- 项目中已有：

```text
go.mod
go.sum
cmd/server/
Makefile
.github/workflows/ci.yml
```

检查 Docker：

```bash
docker version
docker info
docker buildx version
```

## 学完后你应该能做到

- 为 Go 服务编写多阶段 Dockerfile。
- 编写 `.dockerignore` 减少构建上下文。
- 本地构建、运行、查看和删除镜像。
- 设计镜像 tag 策略。
- 把镜像推送到 GHCR/GitLab Registry/Docker Hub。
- 在 GitHub Actions 中构建并推送镜像。
- 上传或保存 SBOM。
- 使用 Trivy 做基础镜像扫描。
- 理解 digest 和不可变部署的关系。

## 推荐官方资料

- Docker Go 指南：<https://docs.docker.com/guides/golang/>
- Dockerfile 最佳实践：<https://docs.docker.com/build/building/best-practices/>
- Docker 多阶段构建：<https://docs.docker.com/build/building/multi-stage/>
- Docker GitHub Actions：<https://docs.docker.com/build/ci/github-actions/>
- GitHub Container Registry：<https://docs.github.com/packages/working-with-a-github-packages-registry/working-with-the-container-registry>
- GitLab Container Registry：<https://docs.gitlab.com/user/packages/container_registry/>
- Trivy：<https://trivy.dev/>

