# 08：环境、AppProject、RBAC 与密钥

## 1. 本节目标

GitOps 会让部署变得透明，但也要设计权限边界。

这一节学习：

- AppProject。
- RBAC。
- environment 权限。
- repo 权限。
- secret 管理边界。

## 2. AppProject 是什么

AppProject 用于限制一组 Applications 可以做什么。

它可以限制：

- 允许的 Git 仓库。
- 允许部署的集群。
- 允许部署的 namespace。
- 允许创建的 Kubernetes 资源类型。

default project 适合学习，但生产应该创建明确 project。

## 3. AppProject 示例

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: go-cicd-lab
  namespace: argocd
spec:
  description: go-cicd-lab applications
  sourceRepos:
    - https://github.com/your-name/go-cicd-lab-deploy.git
  destinations:
    - server: https://kubernetes.default.svc
      namespace: go-cicd-lab-staging
    - server: https://kubernetes.default.svc
      namespace: go-cicd-lab-production
  clusterResourceWhitelist:
    - group: ""
      kind: Namespace
  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"
```

Application 中使用：

```yaml
spec:
  project: go-cicd-lab
```

## 4. 为什么要限制 destinations

如果不限制，错误 Application 可能把资源部署到不该去的 namespace。

例如 staging 应用误部署到 production namespace。

AppProject 可以降低这种风险。

## 5. RBAC

Argo CD RBAC 控制谁能：

- 查看 Application。
- sync Application。
- 修改 Application。
- 删除 Application。
- 管理 repositories。
- 管理 clusters。

生产中应区分：

- 只读用户。
- staging 发布者。
- production 审批者。
- 平台管理员。

学习阶段先理解，不必马上配置复杂规则。

## 6. Git 权限

GitOps 的权限核心转移到 Git。

你需要保护：

- 部署仓库 main 分支。
- production values 文件。
- Application manifests。
- AppProject manifests。

建议：

- production 目录需要 CODEOWNERS。
- production PR 需要 reviewer。
- 禁止直接 push main。
- CI token 最小权限。

## 7. Secret 管理

不要把真实 secret 明文放进 Git。

可选方案：

- 手动预创建 Kubernetes Secret。
- External Secrets Operator。
- Sealed Secrets。
- SOPS + age/GPG。
- 云 Secret Manager。

学习阶段可以：

```bash
kubectl -n go-cicd-lab-staging create secret generic go-cicd-lab-secret ...
```

并让 Helm Chart 只引用 Secret 名称。

## 8. Argo CD 仓库凭证

私有 Git 仓库需要 Argo CD 访问凭证。

建议：

- 使用只读 deploy key。
- 使用最小权限 token。
- 定期轮换。
- 不和个人 token 混用。

生产环境不要让 Argo CD 使用过宽权限的 Git token。

## 9. 小练习

1. 创建 AppProject。
2. 把 staging Application 从 default project 改到新 project。
3. 尝试把 Application destination 改到未授权 namespace，观察错误。
4. 给部署仓库 production 目录加 CODEOWNERS。
5. 写下你的 secret 管理策略。

## 10. 本节小结

你现在应该理解：

- AppProject 限制 Application 的来源和目标。
- RBAC 限制用户对 Argo CD 的操作。
- Git 分支保护是 GitOps 权限核心。
- Secret 不应明文进入部署仓库。
- 私有仓库凭证要最小权限。

