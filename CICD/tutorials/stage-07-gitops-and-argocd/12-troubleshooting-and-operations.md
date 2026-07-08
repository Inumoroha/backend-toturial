# 12：Argo CD 排障与运维

## 1. 本节目标

这一节学习 Argo CD 常见问题排查。

排障要分层：

```text
Git 仓库
-> Argo CD repo-server
-> Application 渲染
-> Kubernetes API
-> 资源健康状态
-> 应用运行状态
```

## 2. 总览命令

```bash
argocd app list
argocd app get go-cicd-lab-staging
argocd app diff go-cicd-lab-staging
argocd app sync go-cicd-lab-staging
argocd app wait go-cicd-lab-staging --sync --health
```

Kubernetes：

```bash
kubectl -n argocd get pods
kubectl -n argocd logs deploy/argocd-server
kubectl -n argocd logs deploy/argocd-repo-server
kubectl -n argocd logs deploy/argocd-application-controller
```

## 3. Repository 访问失败

症状：

```text
repository not accessible
authentication required
```

检查：

- repoURL 是否正确。
- 仓库是否私有。
- Argo CD 是否配置 repo credential。
- token/deploy key 是否过期。
- GitHub/GitLab 权限是否只读即可。

## 4. Helm 渲染失败

症状：

```text
Manifest generation error
```

本地复现：

```bash
helm template go-cicd-lab charts/go-cicd-lab \
  -f environments/staging/values.yaml
```

检查：

- valueFiles 路径是否正确。
- values 类型是否匹配模板。
- 模板缩进是否正确。
- required 值是否缺失。

## 5. Sync 失败

查看：

```bash
argocd app get go-cicd-lab-staging
```

常见原因：

- namespace 无权限创建。
- Secret 不存在。
- imagePullSecret 不存在。
- Kubernetes immutable field 被修改。
- CRD 不存在。
- AppProject 不允许目标 namespace。

## 6. Healthy 失败

Application Synced 但 Degraded：

```text
资源已按 Git 应用，但运行状态不健康。
```

排查：

```bash
kubectl -n go-cicd-lab-staging get pods
kubectl -n go-cicd-lab-staging describe pod <pod>
kubectl -n go-cicd-lab-staging logs <pod>
```

常见：

- ImagePullBackOff。
- CrashLoopBackOff。
- readinessProbe 失败。
- 资源不足。

## 7. OutOfSync 一直存在

可能原因：

- 集群有 webhook 修改资源。
- controller 自动添加字段。
- 手动修改未恢复。
- Helm 模板每次渲染产生随机值。
- 忽略差异规则未配置。

先用：

```bash
argocd app diff go-cicd-lab-staging
```

看具体字段。

不要一上来就忽略差异。先确认差异是否合理。

## 8. 自动同步没有发生

检查：

- Application 是否开启 automated。
- Argo CD 是否能访问 repo。
- targetRevision 是否正确。
- 变更是否已经 push 到目标分支。
- Application 是否被 sync window 限制。
- 是否有 previous sync 失败卡住。

## 9. AppProject 拦截

如果出现 project 权限错误，检查：

```bash
kubectl -n argocd get appproject go-cicd-lab -o yaml
```

重点：

- `sourceRepos`
- `destinations`
- resource whitelist/blacklist

## 10. 紧急处理原则

如果生产事故必须手动修：

1. 记录手动操作。
2. 快速恢复服务。
3. 立刻把 Git 修正到实际期望状态。
4. 确认 Argo CD 回到 Synced。
5. 复盘为什么需要绕过 GitOps。

不要让集群长期处于 Git 无法解释的状态。

## 11. 小练习

故意制造：

1. 错误 repoURL。
2. 错误 valueFiles 路径。
3. 错误 image tag。
4. 手动 scale 造成 drift。
5. AppProject 不允许 namespace。

每次记录：

- Argo CD 状态。
- 关键错误。
- 修复方式。

## 12. 本节小结

你现在应该理解：

- Argo CD 排障要分 Git、渲染、同步、健康几个层级。
- repo-server 负责拉取和渲染。
- application-controller 负责比较和同步。
- OutOfSync 不等于不健康，Degraded 也不等于 Git 不一致。
- 手动救火后必须回写 Git。

