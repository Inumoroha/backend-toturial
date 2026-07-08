# 07：镜像 Tag 更新与 CI 集成

## 1. 本节目标

GitOps 不代表 CI 消失。

这一节学习 CI 和 Argo CD 的分工：

```text
CI 负责构建镜像并更新 Git。
Argo CD 负责从 Git 同步到集群。
```

## 2. 第 6 阶段的方式

CI 直接部署：

```bash
helm upgrade --install ... --set image.tag=sha-8f3a2c1
```

问题：

```text
集群变了，但 Git 中的 image tag 可能没变。
```

## 3. GitOps 的方式

CI 更新部署仓库：

```yaml
image:
  tag: sha-8f3a2c1
```

然后：

```text
commit -> push/PR -> Argo CD sync
```

Git 中能看到每次发布变更。

## 4. staging 自动更新

推荐流程：

```text
app repo main push
-> CI test/lint
-> build image sha-xxxx
-> push image
-> update deploy repo environments/staging/values.yaml
-> commit to deploy repo main
-> Argo CD auto sync staging
```

staging 可以自动 commit。

## 5. production 通过 PR 更新

推荐流程：

```text
app repo tag v0.1.0
-> CI build/push image v0.1.0
-> create deploy repo PR
   production image.tag: v0.1.0
-> review
-> merge
-> Argo CD sync production
```

生产发布变成一个部署仓库 PR。

这很清晰：

- 谁发布。
- 发布哪个版本。
- 改了哪些环境配置。
- 谁审批。

## 6. 用 yq 更新 values

安装 yq 后：

```bash
yq -i '.image.tag = "sha-8f3a2c1"' environments/staging/values.yaml
```

也可以更新 repository：

```bash
yq -i '.image.repository = "ghcr.io/your-name/go-cicd-lab"' environments/staging/values.yaml
```

不要用脆弱的 sed 去改复杂 YAML。

## 7. GitHub Actions 更新部署仓库示例

示例思路：

```yaml
name: update-deploy-repo

on:
  workflow_dispatch:
    inputs:
      image_tag:
        required: true

jobs:
  update-staging:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout deploy repo
        uses: actions/checkout@v6
        with:
          repository: your-name/go-cicd-lab-deploy
          token: ${{ secrets.DEPLOY_REPO_TOKEN }}

      - name: Install yq
        uses: mikefarah/yq@master

      - name: Update image tag
        run: yq -i '.image.tag = "${{ inputs.image_tag }}"' environments/staging/values.yaml

      - name: Commit and push
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add environments/staging/values.yaml
          git commit -m "deploy: staging ${{ inputs.image_tag }}"
          git push
```

注意：

- `DEPLOY_REPO_TOKEN` 需要最小权限。
- 如果没有变化，`git commit` 会失败，需要脚本处理 no-op。
- 生产建议创建 PR，而不是直接 push。

## 8. 创建 production PR

可以用 GitHub CLI：

```bash
gh pr create \
  --title "deploy: production v0.1.0" \
  --body "Deploy go-cicd-lab v0.1.0 to production" \
  --base main \
  --head deploy-production-v0.1.0
```

CI 中需要：

```text
GH_TOKEN
```

并限制权限。

## 9. Argo CD Image Updater

Argo CD 生态中有 Argo CD Image Updater，可以自动发现镜像新版本并更新 Git 或 Application。

初学阶段建议先手写 CI 更新部署仓库。

原因：

- 更清楚 GitOps 基本链路。
- 更容易理解 production PR。
- 不引入额外组件。

等你熟悉后，再研究 Image Updater。

## 10. 小练习

1. 在部署仓库 staging values 中手动改 image tag。
2. commit 并 push。
3. 观察 Argo CD 同步。
4. 用 yq 自动修改 values。
5. 为 production 创建一个 image tag 更新 PR。

## 11. 本节小结

你现在应该理解：

- GitOps 中 image tag 要写入部署仓库。
- CI 负责构建镜像并更新 Git。
- Argo CD 负责同步 Git 到集群。
- staging 可以自动更新，production 推荐 PR 审批。
- yq 比 sed 更适合修改 YAML。

