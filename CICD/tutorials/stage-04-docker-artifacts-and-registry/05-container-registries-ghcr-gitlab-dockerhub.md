# 05：镜像仓库：GHCR、GitLab Registry 与 Docker Hub

## 1. 本节目标

本地镜像只能在本机使用。

要让 CI、服务器、Kubernetes 拉取镜像，你需要把镜像推送到 registry。

这一节学习：

- login。
- tag。
- push。
- pull。
- 常见镜像仓库差异。

## 2. 基本流程

```text
docker build
-> docker tag
-> docker login
-> docker push
-> docker pull
```

示例：

```bash
docker build -t go-cicd-lab:local .
docker tag go-cicd-lab:local ghcr.io/your-name/go-cicd-lab:v0.1.0
docker login ghcr.io
docker push ghcr.io/your-name/go-cicd-lab:v0.1.0
docker pull ghcr.io/your-name/go-cicd-lab:v0.1.0
```

## 3. GitHub Container Registry

GHCR 地址：

```text
ghcr.io
```

镜像名：

```text
ghcr.io/owner/repo:v0.1.0
```

本地登录通常使用 GitHub Personal Access Token：

```bash
echo "$CR_PAT" | docker login ghcr.io -u your-github-username --password-stdin
```

推送：

```bash
docker tag go-cicd-lab:local ghcr.io/your-name/go-cicd-lab:v0.1.0
docker push ghcr.io/your-name/go-cicd-lab:v0.1.0
```

在 GitHub Actions 中，通常可以使用 `GITHUB_TOKEN` 推送当前仓库关联的 package，但 workflow 需要：

```yaml
permissions:
  contents: read
  packages: write
```

## 4. GitLab Container Registry

GitLab registry 常见地址：

```text
registry.gitlab.com
```

镜像名：

```text
registry.gitlab.com/group/project/go-cicd-lab:v0.1.0
```

本地：

```bash
docker login registry.gitlab.com
docker tag go-cicd-lab:local registry.gitlab.com/group/project/go-cicd-lab:v0.1.0
docker push registry.gitlab.com/group/project/go-cicd-lab:v0.1.0
```

GitLab CI 中通常使用内置变量：

```text
CI_REGISTRY
CI_REGISTRY_IMAGE
CI_REGISTRY_USER
CI_REGISTRY_PASSWORD
```

## 5. Docker Hub

Docker Hub 默认 registry 是：

```text
docker.io
```

镜像名：

```text
your-dockerhub-name/go-cicd-lab:v0.1.0
```

登录：

```bash
docker login
```

推送：

```bash
docker tag go-cicd-lab:local your-dockerhub-name/go-cicd-lab:v0.1.0
docker push your-dockerhub-name/go-cicd-lab:v0.1.0
```

CI 中不要写明文密码，使用 secret 或 OIDC/平台推荐方式。

## 6. 私有镜像和公开镜像

公开镜像：

- 任何人可拉取。
- 适合开源项目。
- 不适合包含内部业务服务。

私有镜像：

- 需要认证才能拉取。
- 适合公司内部服务。
- 部署环境需要配置 image pull secret 或平台凭证。

学习项目可以用私有，也可以公开。真实业务服务默认私有。

## 7. 镜像名要小写

很多 registry 要求镜像名小写。

推荐：

```text
ghcr.io/your-name/go-cicd-lab
```

避免：

```text
ghcr.io/YourName/Go-CICD-Lab
```

GitHub Actions 中可以把仓库名转成小写，后面实战会演示。

## 8. 推送前先确认 tag

查看本地镜像：

```bash
docker images
```

查看某个镜像：

```bash
docker images go-cicd-lab
```

推错 tag 是常见事故。

## 9. 删除远程镜像

不同平台删除方式不同，通常在 Web UI 或 API 中操作。

注意：

- 不要随意删除已经部署过的生产镜像。
- 镜像保留策略要考虑回滚窗口。
- 可以清理旧 PR 镜像或临时 tag。

## 10. 小练习

任选一个 registry，完成：

```bash
docker build -t go-cicd-lab:local .
docker tag go-cicd-lab:local <registry>/<namespace>/go-cicd-lab:dev
docker login <registry>
docker push <registry>/<namespace>/go-cicd-lab:dev
docker pull <registry>/<namespace>/go-cicd-lab:dev
```

记录：

- registry 地址。
- 镜像完整名称。
- 是否公开。
- 谁有 push 权限。
- 谁有 pull 权限。

## 11. 本节小结

你现在应该理解：

- registry 用于保存和分发镜像。
- 推送镜像前必须正确 tag。
- GHCR、GitLab Registry、Docker Hub 命名略有不同。
- CI 中推送镜像需要最小权限和 secret 管理。
- 生产镜像不要随便删除。

