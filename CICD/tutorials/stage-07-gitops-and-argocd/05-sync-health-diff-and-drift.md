# 05：Sync、Health、Diff 与 Drift

## 1. 本节目标

这一节学习 Argo CD 状态语言。

你要理解：

- Synced。
- OutOfSync。
- Healthy。
- Progressing。
- Degraded。
- Diff。
- Drift。

## 2. Sync Status

Sync Status 表示 Git 期望状态和集群实际状态是否一致。

常见：

```text
Synced
OutOfSync
Unknown
```

Synced：

```text
集群资源和 Git 渲染结果一致。
```

OutOfSync：

```text
集群实际资源和 Git 不一致。
```

## 3. Health Status

Health Status 表示资源运行健康程度。

常见：

```text
Healthy
Progressing
Degraded
Suspended
Missing
Unknown
```

例如：

- Deployment rollout 完成：Healthy。
- 新 Pod 还没 ready：Progressing。
- Pod CrashLoopBackOff：Degraded。
- Git 中有资源但集群里没有：Missing。

## 4. Sync 和 Health 的区别

可能出现：

```text
Synced + Degraded
```

意思：

```text
集群配置和 Git 一致，但应用运行不健康。
```

也可能：

```text
OutOfSync + Healthy
```

意思：

```text
应用当前运行健康，但集群状态和 Git 不一致。
```

两者都要看。

## 5. Diff

查看差异：

```bash
argocd app diff go-cicd-lab-staging
```

UI 中也可以看 Diff。

Diff 能告诉你：

- Git 中是什么。
- 集群中是什么。
- 哪些字段不同。

## 6. 制造 Drift

手动改副本数：

```bash
kubectl -n go-cicd-lab-staging scale deployment go-cicd-lab-staging --replicas=5
```

然后查看：

```bash
argocd app get go-cicd-lab-staging
argocd app diff go-cicd-lab-staging
```

你应该看到 OutOfSync。

如果没有开启 selfHeal，集群可能保持手动改后的状态，直到你 sync。

## 7. 修复 Drift

手动 sync：

```bash
argocd app sync go-cicd-lab-staging
```

Argo CD 会把副本数改回 Git 中声明的值。

## 8. 常见 OutOfSync 原因

- 手动 kubectl 改了资源。
- CI 修改了集群但没改 Git。
- Kubernetes controller 自动添加字段。
- Helm 模板渲染变化。
- Secret 或 webhook 修改资源。
- 忽略规则未配置。

## 9. 不要在 GitOps 环境手动改集群

紧急情况可以手动操作，但事后必须把 Git 修正回来。

原则：

```text
集群状态最终要回到 Git 可解释的状态。
```

## 10. 小练习

1. 查看 Application 状态。
2. 手动 scale Deployment。
3. 观察 OutOfSync。
4. 查看 diff。
5. 手动 sync 修复。
6. 记录 Argo CD UI 中状态变化。

## 11. 本节小结

你现在应该理解：

- Sync 表示 Git 和集群是否一致。
- Health 表示应用是否健康。
- Diff 用于看差异。
- Drift 是实际状态偏离 Git。
- GitOps 环境中手动改集群要谨慎。

