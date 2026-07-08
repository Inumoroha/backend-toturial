# 13：复习清单与自测题

## 1. 本节目标

用清单和问题确认你是否掌握第 7 阶段。

如果你能用 Argo CD 从 Git 同步 Helm 应用，并能通过 Git 控制发布和回滚，就可以进入第 8 阶段：CI/CD 安全。

## 2. 操作清单

### GitOps 基础

- [ ] 我能解释 GitOps 是什么。
- [ ] 我能区分 push-based CD 和 pull-based GitOps。
- [ ] 我知道 Git 是期望状态来源。
- [ ] 我知道 drift 是什么。
- [ ] 我知道 CI 在 GitOps 中仍负责构建镜像。

### Argo CD 基础

- [ ] 我能安装 Argo CD。
- [ ] 我能访问 Argo CD UI。
- [ ] 我能使用 argocd CLI。
- [ ] 我能创建 Application。
- [ ] 我能手动 sync。
- [ ] 我能查看 app diff。
- [ ] 我能判断 Synced、OutOfSync、Healthy、Degraded。

### 自动同步

- [ ] 我知道 automated sync 的作用。
- [ ] 我知道 prune 的风险。
- [ ] 我知道 selfHeal 的作用。
- [ ] 我能配置 CreateNamespace。
- [ ] 我知道 staging 和 production 自动化程度可以不同。

### 仓库与发布

- [ ] 我能设计应用仓库和部署仓库。
- [ ] 我知道 image tag 要写入部署仓库。
- [ ] 我能用 yq 修改 values。
- [ ] 我知道 staging 可以自动更新。
- [ ] 我知道 production 推荐通过部署仓库 PR。
- [ ] 我能用 Git revert 回滚部署。

### 权限与密钥

- [ ] 我知道 AppProject 的作用。
- [ ] 我知道 RBAC 的作用。
- [ ] 我知道 production values 需要分支保护。
- [ ] 我知道真实 Secret 不应明文进 Git。
- [ ] 我知道私有 repo 凭证要最小权限。

## 3. 自测题

### 题目 1：GitOps 和传统 CI 直接部署的最大区别是什么？

参考答案：

```text
传统 CI 直接连接目标环境并执行部署命令。
GitOps 中 CI 更新 Git 中的期望状态，Argo CD 在集群内读取 Git 并同步集群。
Git 成为部署事实来源。
```

### 题目 2：OutOfSync 和 Degraded 有什么区别？

参考答案：

```text
OutOfSync 表示 Git 期望状态和集群实际状态不一致。
Degraded 表示资源运行状态不健康。
一个应用可以 Synced 但 Degraded，也可以 OutOfSync 但暂时 Healthy。
```

### 题目 3：为什么 GitOps 中 image tag 要写入 Git？

参考答案：

```text
因为 Git 是期望状态来源。
如果 image tag 只通过命令传给集群而不写入 Git，Git 就无法准确描述当前部署版本，审计和回滚都会变差。
```

### 题目 4：production 为什么适合通过部署仓库 PR 发布？

参考答案：

```text
部署仓库 PR 能清楚展示 production image tag 和配置变化。
它可以接受 review、CODEOWNERS、分支保护和审批，Git 历史也会记录谁在什么时候发布了哪个版本。
```

### 题目 5：prune 有什么风险？

参考答案：

```text
prune 会删除 Git 中不再存在的资源。
如果误删了 Git 文件或模板渲染错误，Argo CD 可能删除集群资源。
production 开启 prune 前必须有严格 review 和恢复策略。
```

### 题目 6：selfHeal 有什么风险？

参考答案：

```text
selfHeal 会自动把手动 drift 改回 Git 状态。
这能防止配置漂移，但紧急手动修复如果没有及时写回 Git，可能会被 Argo CD 覆盖。
```

### 题目 7：AppProject 解决什么问题？

参考答案：

```text
AppProject 限制 Application 可以从哪些仓库读取、可以部署到哪些集群和 namespace、可以管理哪些资源类型。
它能降低误部署和越权风险。
```

### 题目 8：GitOps 回滚为什么首选 Git revert？

参考答案：

```text
Git 是事实来源。
如果只在 Argo CD 或 Helm 中手动回滚，Git 仍然指向错误版本，下一次同步可能又把错误版本部署回来。
Git revert 能让期望状态也回到旧版本。
```

## 4. 情景题

### 情景 1

有人手动 `kubectl scale` 把 production 副本从 3 改成 1。Argo CD 显示 OutOfSync。

你会怎么处理？

参考思路：

```text
先确认这是不是紧急变更。
如果不是，手动 sync 或等待 selfHeal 让集群回到 Git 中的 replicas=3。
如果确实要改成 1，必须提交部署仓库 PR 修改 production values。
```

### 情景 2

CI 已经推送了 `v0.2.0` 镜像，但 production 没变化。

你会检查什么？

参考思路：

```text
检查部署仓库 production values 是否已改成 v0.2.0。
检查 PR 是否合并。
检查 Argo CD Application targetRevision 是否正确。
检查是否需要手动 sync。
检查 Argo CD 是否能访问部署仓库。
```

### 情景 3

Argo CD Application 是 Synced，但用户访问服务失败。

你会怎么理解？

参考思路：

```text
Synced 只说明集群资源和 Git 一致，不代表业务可用。
继续看 Health 状态、Pod 状态、Service endpoints、Ingress、应用日志、readinessProbe 和外部依赖。
```

## 5. 第 7 阶段最终作业

完成：

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

应用仓库完成：

```text
.github/workflows/update-deploy-repo.yml
```

要求：

- Argo CD 可以同步 staging。
- staging 开启 automated sync 和 selfHeal。
- production 使用独立 Application。
- production values 通过 PR 修改。
- Secret 不明文提交。
- 可以通过 Git revert 回滚。
- 可以解释 OutOfSync 和 Degraded。

## 6. 进入下一阶段的标准

当你能做到下面几件事，就可以进入第 8 阶段：

1. 能解释 GitOps 和 Argo CD 的工作方式。
2. 能创建和同步 Argo CD Application。
3. 能让 Argo CD 部署 Helm Chart。
4. 能通过部署仓库修改 image tag 触发发布。
5. 能处理 drift 和 sync 失败。
6. 能用 Git revert 回滚。
7. 能说明 GitOps 权限和 secret 风险。

第 8 阶段会系统学习 CI/CD 安全：secret 管理、最小权限、OIDC、镜像签名、SBOM、provenance 和 SLSA。

