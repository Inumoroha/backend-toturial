# 阶段 7：GitOps 与 Argo CD

> 本阶段目标：把第 6 阶段的“CI 直接执行 Helm 部署”升级为 GitOps 模式，让 Git 仓库保存期望状态，由 Argo CD 在集群内持续同步。

## 学习顺序

请按下面顺序学习：

1. [01-gitops-mental-model.md](./01-gitops-mental-model.md)
   - 理解 GitOps、期望状态、实际状态、drift。
   - 区分 push-based CD 和 pull-based GitOps。

2. [02-repository-structure-for-gitops.md](./02-repository-structure-for-gitops.md)
   - 设计应用仓库和部署仓库。
   - 学习环境目录、Helm values、image tag 更新策略。

3. [03-install-argocd-locally.md](./03-install-argocd-locally.md)
   - 在 kind/minikube 安装 Argo CD。
   - 访问 UI，安装 CLI，登录 Argo CD。

4. [04-first-argocd-application.md](./04-first-argocd-application.md)
   - 创建第一个 Argo CD Application。
   - 让 Argo CD 从 Git 仓库同步 Helm Chart 到 Kubernetes。

5. [05-sync-health-diff-and-drift.md](./05-sync-health-diff-and-drift.md)
   - 学习 Sync、OutOfSync、Healthy、Degraded、Diff。
   - 观察手动改集群资源后的 drift。

6. [06-auto-sync-prune-self-heal-and-sync-options.md](./06-auto-sync-prune-self-heal-and-sync-options.md)
   - 学习自动同步、prune、selfHeal、syncOptions。
   - 理解自动化带来的收益和风险。

7. [07-image-tag-update-and-ci-integration.md](./07-image-tag-update-and-ci-integration.md)
   - 学习 CI 和 GitOps 的分工。
   - 镜像构建后通过 PR 更新部署仓库中的 image tag。

8. [08-environments-projects-rbac-and-secrets.md](./08-environments-projects-rbac-and-secrets.md)
   - 学习 AppProject、RBAC、environment 权限边界。
   - 理解 GitOps 中密钥不应该明文进 Git。

9. [09-rollbacks-history-and-release-promotion.md](./09-rollbacks-history-and-release-promotion.md)
   - 学习 Git revert、Argo CD rollback、环境晋级。
   - 理解 GitOps 回滚和 Helm rollback 的差异。

10. [10-application-set-and-app-of-apps.md](./10-application-set-and-app-of-apps.md)
    - 了解 ApplicationSet 和 App of Apps。
    - 为多服务、多环境管理建立概念。

11. [11-practice-gitops-for-go-cicd-lab.md](./11-practice-gitops-for-go-cicd-lab.md)
    - 为 `go-cicd-lab` 落地 GitOps 部署仓库、Argo CD Application、CI 更新 image tag。
    - 输出 `docs/gitops.md`。

12. [12-troubleshooting-and-operations.md](./12-troubleshooting-and-operations.md)
    - 学习 Argo CD 常见问题排查。
    - 处理 repo 访问失败、sync 失败、health 异常、权限问题。

13. [13-review-checklist-and-quiz.md](./13-review-checklist-and-quiz.md)
    - 用清单和自测题检查是否可以进入第 8 阶段。

## 本阶段建议时间

- 快速学习：1 周。
- 扎实学习：2 周。
- 推荐方式：先在本地 kind/minikube 安装 Argo CD，再把第 6 阶段的 Helm Chart 接入 GitOps。

## 本阶段你要准备什么

- 已完成第 6 阶段：能用 Helm 部署 Go 服务。
- 一个 Kubernetes 集群，推荐本地 kind/minikube。
- 一个应用仓库，例如 `go-cicd-lab`。
- 一个部署仓库，例如 `go-cicd-lab-deploy`。
- kubectl、helm、argocd CLI。

检查：

```bash
kubectl version --client
helm version
argocd version --client
```

## 学完后你应该能做到

- 解释 GitOps 的工作方式。
- 安装并访问 Argo CD。
- 用 Application 从 Git 同步 Helm Chart。
- 判断 Synced、OutOfSync、Healthy、Degraded。
- 配置自动同步、prune、selfHeal。
- 让 CI 构建镜像后更新部署仓库。
- 用 Git PR 控制 production 发布。
- 处理常见 Argo CD 同步失败。

## 推荐官方资料

- Argo CD Overview：<https://argo-cd.readthedocs.io/>
- Argo CD Getting Started：<https://argo-cd.readthedocs.io/en/latest/getting_started/>
- Argo CD Declarative Setup：<https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/>
- Argo CD Helm：<https://argo-cd.readthedocs.io/en/latest/user-guide/helm/>
- Argo CD Automated Sync：<https://argo-cd.readthedocs.io/en/latest/user-guide/auto_sync/>
- Argo CD Sync Options：<https://argo-cd.readthedocs.io/en/latest/user-guide/sync-options/>
- Argo CD Projects：<https://argo-cd.readthedocs.io/en/stable/user-guide/projects/>
- Argo CD RBAC：<https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/>

