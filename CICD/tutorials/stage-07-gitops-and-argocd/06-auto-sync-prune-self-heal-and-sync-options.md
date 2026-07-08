# 06：自动同步、Prune、Self Heal 与 Sync Options

## 1. 本节目标

上一节你手动 sync。

这一节学习自动化：

- automated sync。
- prune。
- selfHeal。
- syncOptions。
- 自动同步的风险边界。

## 2. 手动同步

手动同步模式：

```text
Git changed
-> Argo CD shows OutOfSync
-> human clicks Sync or runs argocd app sync
```

优点：

- 可控。
- 适合 production 初期。

缺点：

- 需要人工操作。
- 容易忘记同步。

## 3. 自动同步

Application 中开启：

```yaml
spec:
  syncPolicy:
    automated: {}
```

含义：

```text
Argo CD 发现 Git 变化后自动同步。
```

staging 推荐开启。

production 是否开启取决于团队成熟度。即使开启，也通常通过部署仓库 PR 控制生产变更。

## 4. Prune

如果 Git 中删除了某个资源，默认不一定会自动删除集群中的资源。

开启 prune：

```yaml
spec:
  syncPolicy:
    automated:
      prune: true
```

含义：

```text
Git 中不再存在的资源，Argo CD 可以从集群删除。
```

风险：

- 错删 Git 文件可能导致集群资源被删。
- production 开启前要有 review 和备份意识。

## 5. Self Heal

开启 selfHeal：

```yaml
spec:
  syncPolicy:
    automated:
      selfHeal: true
```

含义：

```text
如果有人手动改集群导致 drift，Argo CD 自动改回 Git 状态。
```

好处：

- 防止手动漂移。
- 强化 Git 事实来源。

风险：

- 紧急手动修复可能被自动覆盖。
- 团队必须形成“改 Git 而不是改集群”的习惯。

## 6. 推荐 staging 配置

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - CreateNamespace=true
```

staging 自动化程度可以高一些。

## 7. 推荐 production 配置

初学阶段 production 可以：

```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true
```

也就是不自动同步，由人工点 Sync。

成熟后可以：

```yaml
syncPolicy:
  automated:
    prune: false
    selfHeal: true
  syncOptions:
    - CreateNamespace=true
```

是否开启 prune 要谨慎。

## 8. 常见 Sync Options

常用：

```yaml
syncOptions:
  - CreateNamespace=true
  - PrunePropagationPolicy=foreground
  - PruneLast=true
```

说明：

- `CreateNamespace=true`：目标 namespace 不存在时创建。
- `PruneLast=true`：先应用新资源，最后删除旧资源。
- `PrunePropagationPolicy`：控制删除传播策略。

初学阶段重点掌握 `CreateNamespace=true`。

## 9. sync wave

Argo CD 支持通过 annotation 控制同步顺序：

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

例如：

```text
Secret/ConfigMap wave 0
Migration Job wave 1
Deployment wave 2
```

初学阶段先知道有这个能力，不要过度使用。

## 10. 小练习

1. 给 staging Application 开启 automated sync。
2. 修改 staging values 中 replicaCount。
3. 合并后观察 Argo CD 自动同步。
4. 手动 kubectl scale Deployment。
5. 开启 selfHeal 后观察是否自动恢复。
6. 从 Git 删除一个测试 ConfigMap，观察 prune 行为。

## 11. 本节小结

你现在应该理解：

- automated sync 让 Git 变化自动进入集群。
- prune 会删除 Git 中已移除的资源。
- selfHeal 会修复手动 drift。
- staging 可以更自动，production 要更谨慎。
- 自动化越强，Git review 越重要。

