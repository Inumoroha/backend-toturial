# 14：复习清单与自测题

## 1. 本节目标

用清单和问题确认你是否掌握第 6 阶段。

如果你能用 Helm 把 Go 服务部署到 Kubernetes，并能排查和回滚，就可以进入第 7 阶段：GitOps 与 Argo CD。

## 2. 操作清单

### Kubernetes 基础

- [ ] 我能解释 Pod、Deployment、Service、Namespace。
- [ ] 我能使用 kubectl context。
- [ ] 我能创建 kind 或 minikube 集群。
- [ ] 我能 apply manifests。
- [ ] 我能 port-forward 访问服务。

### 配置与密钥

- [ ] 我能使用 ConfigMap。
- [ ] 我能使用 Secret。
- [ ] 我知道真实 Secret 不应该提交 Git。
- [ ] 我能创建 imagePullSecret。
- [ ] 我知道配置变化后 Pod 可能需要重启。

### Deployment 运维

- [ ] 我能配置 readinessProbe。
- [ ] 我能配置 livenessProbe。
- [ ] 我能配置 startupProbe。
- [ ] 我能配置 resources requests/limits。
- [ ] 我能查看 rollout status。
- [ ] 我能执行 rollout undo。

### Service 与入口

- [ ] 我知道 ClusterIP、NodePort、LoadBalancer 的区别。
- [ ] 我知道 Ingress 需要 Ingress Controller。
- [ ] 我能查看 Service endpoints。
- [ ] 我能用 port-forward 本地调试。

### Helm

- [ ] 我能解释 Chart、values、template、release。
- [ ] 我能写最小 Chart。
- [ ] 我能使用 `helm lint`。
- [ ] 我能使用 `helm template`。
- [ ] 我能使用 `helm upgrade --install`。
- [ ] 我能使用 `helm history`。
- [ ] 我能使用 `helm rollback`。
- [ ] 我能为 staging/production 写不同 values 文件。

### CI/CD

- [ ] 我能在 PR 中校验 Helm Chart。
- [ ] 我知道 kubeconfig 是敏感凭证。
- [ ] 我知道 production 部署需要审批。
- [ ] 我能用 Helm 部署指定 image tag。

## 3. 自测题

### 题目 1：为什么通常不直接创建 Pod，而是创建 Deployment？

参考答案：

```text
Pod 生命周期短，可能被删除或替换。
Deployment 能管理副本数、滚动更新和回滚，更适合长期运行的无状态服务。
```

### 题目 2：Service selector 错了会发生什么？

参考答案：

```text
Service 找不到后端 Pod，endpoints 为空。
访问 Service 会失败，即使 Pod 本身是 Running。
```

### 题目 3：readinessProbe 和 livenessProbe 有什么区别？

参考答案：

```text
readinessProbe 决定 Pod 是否接收流量。
livenessProbe 决定容器是否需要被重启。
数据库短暂不可用通常不应该让 liveness 失败，但可以让 readiness 失败。
```

### 题目 4：为什么 Secret 仍然不能随便提交到 Git？

参考答案：

```text
Kubernetes Secret YAML 中的数据可以被解码或直接使用。
真实 Secret 进入 Git 后就可能泄漏，应该由 secret manager、CI 环境密钥或受控流程注入。
```

### 题目 5：Helm Chart 的 version 和 appVersion 有什么区别？

参考答案：

```text
version 是 Chart 包本身的版本。
appVersion 是应用版本。
修改模板或 values 结构通常提升 Chart version，发布应用新镜像通常改变 appVersion 或 image tag。
```

### 题目 6：`helm upgrade --install --wait --atomic` 有什么作用？

参考答案：

```text
upgrade --install 表示不存在就安装，存在就升级。
--wait 等待资源 ready。
--atomic 在升级失败时尝试回滚 Kubernetes 资源。
它不能自动回滚数据库迁移。
```

### 题目 7：为什么 image tag 通常通过 `--set image.tag=...` 传入？

参考答案：

```text
每次发布的镜像 tag 都会变化。
把 tag 通过部署命令传入，可以复用同一份 Chart 和环境 values，减少频繁改 values 文件。
```

### 题目 8：Helm 部署失败你会如何排查？

参考答案：

```text
先 helm template 查看渲染结果，再 helm status/history 查看 release 状态。
然后 kubectl get pods、describe pod、logs、events，判断是模板问题、镜像问题、probe 问题还是资源问题。
```

## 4. 情景题

### 情景 1

Pod 状态是 `ImagePullBackOff`。

你会检查什么？

参考思路：

```text
检查镜像名称和 tag 是否存在。
检查镜像是否私有。
检查 imagePullSecret 是否存在且被 Deployment 引用。
检查 registry token 是否有效。
检查 Node 是否能访问 registry。
```

### 情景 2

Deployment 更新后一直卡在 rollout。

你会检查什么？

参考思路：

```text
kubectl rollout status 查看卡住。
kubectl get pods 看新 Pod 状态。
kubectl describe pod 看 Events。
kubectl logs 看应用日志。
重点检查 readinessProbe、镜像、环境变量、Secret、资源不足。
```

### 情景 3

生产 Helm rollback 后应用仍然报数据库字段不存在。

你怎么理解？

参考思路：

```text
Helm rollback 只回滚 Kubernetes 资源，例如 Deployment 镜像和配置。
如果之前执行了破坏性数据库迁移，旧版本应用可能无法兼容当前 schema。
这说明数据库迁移没有遵循向后兼容或 expand/contract。
```

## 5. 第 6 阶段最终作业

在 `go-cicd-lab` 中完成：

```text
deploy/k8s/base/
deploy/helm/go-cicd-lab/
.github/workflows/k8s.yml
docs/kubernetes.md
```

要求：

- kind/minikube 本地部署成功。
- Deployment 使用 readiness/liveness/startup probes。
- Deployment 配置 resources。
- Service 可通过 port-forward 访问。
- ConfigMap 和 Secret 分离配置。
- Helm Chart 支持 staging/production values。
- `helm lint` 和 `helm template` 通过。
- `helm upgrade --install` 成功。
- `helm rollback` 成功。
- CI 能校验 Helm Chart。

## 6. 进入下一阶段的标准

当你能做到下面几件事，就可以进入第 7 阶段：

1. 能解释 Kubernetes 核心对象之间的关系。
2. 能把 Go 服务部署到 Kubernetes。
3. 能用 Helm 管理多环境部署。
4. 能排查 Pod 和 Service 常见问题。
5. 能回滚 Helm release。
6. 能在 CI 中校验 Chart。

第 7 阶段会学习 GitOps 与 Argo CD：让集群控制器从 Git 仓库同步期望状态，而不是让 CI 直接执行部署命令。

