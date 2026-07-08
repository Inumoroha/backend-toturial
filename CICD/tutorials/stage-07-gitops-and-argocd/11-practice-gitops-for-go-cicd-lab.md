# 11：实战，为 go-cicd-lab 落地 GitOps

## 1. 本节目标

这一节把第 7 阶段落地。

最终产出：

```text
go-cicd-lab-deploy/
  charts/go-cicd-lab/
  environments/staging/values.yaml
  environments/staging/application.yaml
  environments/production/values.yaml
  environments/production/application.yaml
  projects/go-cicd-lab-project.yaml
  docs/gitops.md
```

应用仓库新增：

```text
.github/workflows/update-deploy-repo.yml
```

## 2. 创建部署仓库

新建仓库：

```text
go-cicd-lab-deploy
```

目录：

```bash
mkdir -p charts environments/staging environments/production projects docs
```

把第 6 阶段的 Helm Chart 复制到：

```text
charts/go-cicd-lab/
```

## 3. staging values

`environments/staging/values.yaml`：

```yaml
replicaCount: 1

image:
  repository: ghcr.io/your-name/go-cicd-lab
  tag: sha-example
  pullPolicy: Always

config:
  APP_ENV: staging
  HTTP_ADDR: ":8080"
  LOG_LEVEL: debug

secret:
  name: go-cicd-lab-secret

ingress:
  enabled: false
```

## 4. production values

`environments/production/values.yaml`：

```yaml
replicaCount: 3

image:
  repository: ghcr.io/your-name/go-cicd-lab
  tag: v0.1.0
  pullPolicy: IfNotPresent

config:
  APP_ENV: production
  HTTP_ADDR: ":8080"
  LOG_LEVEL: info

secret:
  name: go-cicd-lab-secret

ingress:
  enabled: false
```

真实 production 初始 tag 应该是已经存在的 release 镜像。

## 5. AppProject

`projects/go-cicd-lab-project.yaml`：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: go-cicd-lab
  namespace: argocd
spec:
  description: go-cicd-lab GitOps project
  sourceRepos:
    - https://github.com/your-name/go-cicd-lab-deploy.git
  destinations:
    - server: https://kubernetes.default.svc
      namespace: go-cicd-lab-staging
    - server: https://kubernetes.default.svc
      namespace: go-cicd-lab-production
  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
```

应用：

```bash
kubectl apply -f projects/go-cicd-lab-project.yaml
```

## 6. staging Application

`environments/staging/application.yaml`：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: go-cicd-lab-staging
  namespace: argocd
spec:
  project: go-cicd-lab
  source:
    repoURL: https://github.com/your-name/go-cicd-lab-deploy.git
    targetRevision: main
    path: charts/go-cicd-lab
    helm:
      valueFiles:
        - ../../environments/staging/values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: go-cicd-lab-staging
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## 7. production Application

`environments/production/application.yaml`：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: go-cicd-lab-production
  namespace: argocd
spec:
  project: go-cicd-lab
  source:
    repoURL: https://github.com/your-name/go-cicd-lab-deploy.git
    targetRevision: main
    path: charts/go-cicd-lab
    helm:
      valueFiles:
        - ../../environments/production/values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: go-cicd-lab-production
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

production 先不开自动同步。

你可以在 UI 中人工 sync，或后续成熟后开启。

## 8. 创建 Kubernetes Secret

学习阶段手动创建：

```bash
kubectl create namespace go-cicd-lab-staging --dry-run=client -o yaml | kubectl apply -f -
kubectl -n go-cicd-lab-staging create secret generic go-cicd-lab-secret \
  --from-literal=DATABASE_URL='postgres://user:password@postgres:5432/go_cicd_lab?sslmode=disable' \
  --from-literal=JWT_SECRET='change-me'
```

production 同理，但使用不同值。

不要把真实 Secret 提交到部署仓库。

## 9. 应用 Argo CD 资源

```bash
kubectl apply -f projects/go-cicd-lab-project.yaml
kubectl apply -f environments/staging/application.yaml
kubectl apply -f environments/production/application.yaml
```

查看：

```bash
argocd app list
argocd app get go-cicd-lab-staging
argocd app sync go-cicd-lab-staging
argocd app wait go-cicd-lab-staging --sync --health
```

## 10. 应用仓库更新部署仓库

在 `go-cicd-lab` 应用仓库创建：

```text
.github/workflows/update-deploy-repo.yml
```

示例：

```yaml
name: update-deploy-repo

on:
  workflow_dispatch:
    inputs:
      environment:
        required: true
        type: choice
        options:
          - staging
          - production
      image_tag:
        required: true
        type: string

permissions:
  contents: read

jobs:
  update:
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
        run: |
          yq -i '.image.tag = "${{ inputs.image_tag }}"' \
            environments/${{ inputs.environment }}/values.yaml

      - name: Commit
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add environments/${{ inputs.environment }}/values.yaml
          if git diff --cached --quiet; then
            echo "No changes"
            exit 0
          fi
          git commit -m "deploy: ${{ inputs.environment }} ${{ inputs.image_tag }}"
          git push
```

生产建议改造成创建 PR，而不是直接 push。

## 11. 部署文档

`docs/gitops.md`：

````markdown
# GitOps 部署说明

## 仓库

- 应用仓库：`go-cicd-lab`
- 部署仓库：`go-cicd-lab-deploy`

## 环境

- staging：自动同步，自动修复 drift。
- production：通过 PR 修改 `environments/production/values.yaml`，人工 sync 或审批后同步。

## 手动同步

```bash
argocd app sync go-cicd-lab-staging
argocd app wait go-cicd-lab-staging --sync --health
```

## 更新镜像

```bash
yq -i '.image.tag = "sha-xxxxxxx"' environments/staging/values.yaml
git commit -am "deploy: staging sha-xxxxxxx"
git push
```

## 回滚

```bash
git revert <deploy-commit>
git push
```

然后等待 Argo CD 同步。

## 排障

```bash
argocd app get go-cicd-lab-staging
argocd app diff go-cicd-lab-staging
kubectl -n go-cicd-lab-staging get pods
kubectl -n go-cicd-lab-staging describe pod <pod>
```
````

## 12. 验收

你完成后应该能做到：

- Argo CD 中出现 staging 和 production Applications。
- staging 自动同步。
- production 手动同步。
- 修改 staging image tag 后自动部署。
- 手动 kubectl 改副本数后，staging selfHeal 恢复。
- 回滚通过 Git revert 完成。

## 13. 本节小结

你已经把 `go-cicd-lab` 从 CI/CD 推进到 GitOps。

现在，部署状态由 Git 记录，Argo CD 负责同步，CI 不再直接改 Kubernetes 集群。

