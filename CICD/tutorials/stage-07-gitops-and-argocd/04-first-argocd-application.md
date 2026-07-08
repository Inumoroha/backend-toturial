# 04：第一个 Argo CD Application

## 1. 本节目标

这一节创建第一个 Argo CD Application。

目标：

```text
Argo CD 读取 Git 仓库中的 Helm Chart
-> 渲染 manifests
-> 同步到 Kubernetes namespace
```

## 2. Application 是什么

Application 是 Argo CD 的核心 CRD。

它描述：

- 从哪个 Git 仓库读取。
- 读取哪个 path。
- 用哪个 branch/tag/commit。
- 部署到哪个 cluster。
- 部署到哪个 namespace。
- 是否自动同步。

## 3. 最小 Application

`environments/staging/application.yaml`：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: go-cicd-lab-staging
  namespace: argocd
spec:
  project: default
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
    syncOptions:
      - CreateNamespace=true
```

注意：

- Application 资源通常创建在 Argo CD namespace 中，也就是 `argocd`。
- `repoURL` 换成你的部署仓库。
- `path` 指向 Chart 目录。
- `valueFiles` 路径相对 Chart 目录解析。

## 4. 应用 Application

```bash
kubectl apply -n argocd -f environments/staging/application.yaml
```

查看：

```bash
argocd app list
argocd app get go-cicd-lab-staging
```

或者在 UI 查看。

## 5. 手动同步

```bash
argocd app sync go-cicd-lab-staging
```

等待：

```bash
argocd app wait go-cicd-lab-staging --health --sync
```

查看 Kubernetes：

```bash
kubectl -n go-cicd-lab-staging get all
```

## 6. 使用 UI 创建 Application

UI 也可以创建 Application。

但 GitOps 推荐把 Application 本身也写成 YAML 提交。

原因：

- 可审查。
- 可追踪。
- 可恢复。
- 可在新集群重新 apply。

## 7. private repo 怎么办

私有仓库需要在 Argo CD 中配置仓库凭证。

可以通过 UI、CLI 或 declarative secret 配置。

学习阶段建议先用公开测试仓库，跑通后再配置私有仓库。

生产环境要使用只读 deploy key 或最小权限 token。

## 8. 小练习

1. 创建部署仓库。
2. 放入 Helm Chart 和 staging values。
3. 创建 `application.yaml`。
4. `kubectl apply -n argocd -f application.yaml`。
5. 手动 sync。
6. 通过 port-forward 访问服务。

## 9. 本节小结

你现在应该理解：

- Argo CD Application 描述 Git 到集群的同步关系。
- Application 本身也应该声明式管理。
- 手动 sync 可以把 Git 期望状态应用到集群。
- 同集群 destination server 使用 `https://kubernetes.default.svc`。

