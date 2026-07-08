# 04：部署仓库最终结构

## 1. 本节目标

部署仓库展示的是 CD 和 GitOps 能力。

它要证明你知道：

```text
如何把镜像部署到环境。
如何区分 staging 和 production。
如何通过 Git 管理期望状态。
如何审批、回滚和审计发布。
```

## 2. 推荐目录

```text
go-cicd-lab-deploy/
  charts/
    go-cicd-lab/
      Chart.yaml
      values.yaml
      templates/
        deployment.yaml
        service.yaml
        ingress.yaml
        configmap.yaml
        serviceaccount.yaml
        servicemonitor.yaml
  environments/
    staging/
      values.yaml
    production/
      values.yaml
  applications/
    staging.yaml
    production.yaml
  projects/
    go-cicd-lab-project.yaml
  docs/
    release.md
    rollback.md
    gitops.md
  README.md
```

## 3. Chart 基础信息

`charts/go-cicd-lab/Chart.yaml`：

```yaml
apiVersion: v2
name: go-cicd-lab
description: Helm chart for go-cicd-lab
type: application
version: 0.1.0
appVersion: "0.1.0"
```

默认 `values.yaml`：

```yaml
replicaCount: 1

image:
  repository: ghcr.io/your-org/go-cicd-lab
  tag: ""
  digest: ""
  pullPolicy: IfNotPresent

service:
  port: 80
  targetPort: 8080

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

## 4. Staging 与 Production values

`environments/staging/values.yaml`：

```yaml
replicaCount: 1

image:
  tag: "main"
  digest: ""

env:
  APP_ENV: staging
```

`environments/production/values.yaml`：

```yaml
replicaCount: 3

image:
  tag: ""
  digest: "sha256:replace-me"

env:
  APP_ENV: production
```

建议：

```text
staging 可以使用自动更新。
production 尽量使用 digest，并通过 PR 修改。
```

## 5. Argo CD Application

`applications/staging.yaml`：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: go-cicd-lab-staging
  namespace: argocd
spec:
  project: go-cicd-lab
  source:
    repoURL: https://github.com/your-org/go-cicd-lab-deploy.git
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

`production.yaml` 可以不开自动同步：

```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true
```

## 6. AppProject

`projects/go-cicd-lab-project.yaml`：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: go-cicd-lab
  namespace: argocd
spec:
  sourceRepos:
    - https://github.com/your-org/go-cicd-lab-deploy.git
  destinations:
    - namespace: go-cicd-lab-staging
      server: https://kubernetes.default.svc
    - namespace: go-cicd-lab-production
      server: https://kubernetes.default.svc
  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"
```

生产中可以进一步限制资源类型。

## 7. README 必须回答的问题

部署仓库 README 至少包含：

- 仓库职责。
- 环境说明。
- 如何渲染 Helm。
- 如何部署到 staging。
- 如何发布 production。
- 如何回滚。
- 如何处理 Secret。
- 如何查看 Argo CD 状态。

示例：

```markdown
# go-cicd-lab-deploy

## Environments

## Helm Render

## Staging Release

## Production Release

## Rollback

## Secret Management

## Argo CD Operations
```

## 8. 本地校验命令

渲染：

```bash
helm template go-cicd-lab ./charts/go-cicd-lab \
  -f ./environments/staging/values.yaml
```

lint：

```bash
helm lint ./charts/go-cicd-lab \
  -f ./environments/staging/values.yaml
```

检查 Kubernetes YAML：

```bash
helm template go-cicd-lab ./charts/go-cicd-lab \
  -f ./environments/staging/values.yaml \
  > rendered.yaml
```

## 9. GitOps PR 模板

部署仓库可以增加：

```text
.github/pull_request_template.md
```

内容：

```markdown
## Deployment Change

- Environment:
- Image:
- Digest:
- Source commit:
- Workflow run:

## Checks

- [ ] Helm lint passed.
- [ ] Image signature verified.
- [ ] Smoke test plan exists.
- [ ] Rollback plan exists.
- [ ] Production owner approved.
```

## 10. 小练习

完成：

1. 创建部署仓库目录。
2. 写 Helm chart。
3. 写 staging/production values。
4. 写 Argo CD Application。
5. 写 AppProject。
6. 写部署仓库 README。
7. 本地运行 `helm lint` 和 `helm template`。

## 11. 本节小结

你现在应该理解：

- 部署仓库是 GitOps 的期望状态来源。
- staging 和 production values 要分离。
- production 发布应通过 PR。
- Argo CD Application 和 AppProject 是部署边界。
- 部署仓库文档是面试展示的重要证据。

