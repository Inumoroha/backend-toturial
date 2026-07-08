# 10：Kubernetes、GitOps 与部署安全

## 1. 本节目标

前面已经学习了 Kubernetes、Helm 和 Argo CD。

到了安全阶段，要重新审视部署链路的权限边界：

```text
谁能改部署配置？
谁能访问集群？
谁能修改 production？
Argo CD 能管理哪些资源？
CI 是否还持有 kubeconfig？
Secret 是否明文进 Git？
```

这一节学习：

- Kubernetes 部署权限最小化。
- GitOps 仓库保护。
- Argo CD AppProject/RBAC 加固。
- production 变更审批。
- Secret 管理方式。

## 2. Push CD 与 GitOps 的权限差异

传统 push-based CD：

```text
GitHub Actions 持有 kubeconfig。
CI 直接 kubectl/helm 部署到集群。
```

GitOps：

```text
CI 构建镜像。
CI 更新部署仓库。
Argo CD 在集群内拉取 Git 并同步。
```

安全优势：

- CI 不一定需要 production kubeconfig。
- 部署变更可以通过 Git PR 审批。
- 集群访问权限集中在 Argo CD。
- Git 记录 production 状态变化。

但 GitOps 不是天然安全。

如果部署仓库没有保护，攻击者只要改 values 就能发布恶意镜像。

## 3. kubeconfig 最小权限

如果某些 job 仍需要访问集群，例如 staging 临时部署，kubeconfig 应满足：

```text
只访问目标 namespace。
只允许必要 verbs。
只在必要 job 使用。
不用于 PR from fork。
不用于 production 直接部署。
定期轮换。
```

示例 Role：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: go-cicd-lab-staging
rules:
  - apiGroups: ["", "apps", "batch", "networking.k8s.io"]
    resources:
      - deployments
      - services
      - configmaps
      - secrets
      - jobs
      - ingresses
    verbs: ["get", "list", "watch", "create", "update", "patch"]
```

再绑定给专用 ServiceAccount：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployer
  namespace: go-cicd-lab-staging
subjects:
  - kind: ServiceAccount
    name: deployer
    namespace: go-cicd-lab-staging
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

避免：

```text
cluster-admin kubeconfig 放进 CI secrets。
同一个 kubeconfig 同时用于 staging 和 production。
PR workflow 可以访问 kubeconfig。
```

## 4. GitHub Environments 保护 production

production 发布 workflow 建议使用 GitHub Environments：

```yaml
jobs:
  promote:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: echo "promote to production"
```

在 GitHub 页面设置：

- Required reviewers。
- Wait timer。
- Deployment branches。
- Environment secrets。

效果：

```text
只有符合环境规则的 workflow 才能读取 production secrets。
进入 production 前需要审批。
```

即使使用 GitOps，也可以把“更新 production 部署仓库 PR”放在 environment 审批后。

## 5. 部署仓库保护

部署仓库是 production 期望状态来源，必须保护。

建议：

- `main` 分支开启保护。
- 必须 PR 合并。
- 必须通过 CI 校验。
- 必须 CODEOWNERS review。
- 禁止 force push。
- production 目录由平台/运维负责人审批。

示例 CODEOWNERS：

```text
/environments/production/ @your-org/platform-team
/projects/ @your-org/platform-team
/charts/ @your-org/backend-leads @your-org/platform-team
```

部署仓库 PR 应展示：

```text
image tag/digest 变化
replicas 变化
resource 变化
ingress 变化
secret 引用变化
RBAC 变化
```

## 6. Argo CD AppProject 加固

AppProject 用来限制 Application 的边界。

示例：

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
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"
```

生产中可以进一步收紧：

```text
限制 sourceRepos。
限制 destination namespace。
限制可管理资源类型。
禁止普通应用项目管理 ClusterRole、ClusterRoleBinding。
为 production 使用单独 AppProject。
```

## 7. Argo CD RBAC

Argo CD UI/CLI 也要控制权限。

思路：

```text
开发者可以看应用状态。
开发者可以同步 staging。
production 同步需要平台团队或发布负责人。
只有少数人能修改 AppProject 和 repo credential。
```

不要让所有人都是 Argo CD admin。

## 8. Secret 管理

不要把真实 secret 明文放进 Git。

常见方案：

- External Secrets Operator。
- Sealed Secrets。
- SOPS 加密文件。
- 云厂商 Secret Manager。
- Vault。

GitOps 中可以提交：

```text
secret 引用
加密后的 secret
ExternalSecret manifest
```

但不提交：

```text
数据库明文密码
JWT 私钥
云访问密钥
第三方 API token
```

ExternalSecret 示例：

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: go-cicd-lab
  namespace: go-cicd-lab-production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: production-secret-store
    kind: ClusterSecretStore
  target:
    name: go-cicd-lab
  data:
    - secretKey: DATABASE_URL
      remoteRef:
        key: production/go-cicd-lab/database-url
```

## 9. Production PR 检查项

production 变更至少检查：

- 镜像是否使用 digest。
- 镜像 digest 是否有可信签名。
- SBOM/provenance 是否生成。
- values diff 是否符合预期。
- 是否修改 RBAC。
- 是否新增 Secret 引用。
- 是否修改 Ingress/TLS。
- 是否影响资源配额。
- 是否有回滚方案。

## 10. 小练习

对部署仓库完成：

1. 为 production 目录增加 CODEOWNERS。
2. 为 production Application 使用单独 AppProject。
3. 检查 AppProject 的 sourceRepos 和 destinations。
4. 确认 CI 不再持有 production cluster-admin kubeconfig。
5. 写出 production Secret 管理方案。
6. 在 production PR 模板中加入安全检查项。

## 11. 本节小结

你现在应该理解：

- GitOps 降低了 CI 直接访问集群的需求，但部署仓库必须保护。
- kubeconfig 应最小权限，production 尽量不放在 CI。
- AppProject 限制 Argo CD 可同步范围。
- Argo CD RBAC 控制谁能看、同步和管理应用。
- production Secret 不应明文进入 Git。

