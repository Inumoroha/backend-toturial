# 09：回滚、历史与环境晋级

## 1. 本节目标

GitOps 下的回滚和传统 Helm rollback 有些不同。

这一节学习：

- Git revert 回滚。
- Argo CD rollback。
- 环境晋级。
- release 历史。

## 2. GitOps 推荐回滚方式

推荐首选：

```text
revert 部署仓库 commit
-> merge
-> Argo CD sync
```

例如 production values 从：

```yaml
image:
  tag: v0.1.1
```

回到：

```yaml
image:
  tag: v0.1.0
```

这个变化应该出现在 Git 历史中。

## 3. 为什么 Git revert 更适合 GitOps

因为 Git 是事实来源。

如果你直接在 Argo CD 或 Helm 中回滚集群，但 Git 没变：

```text
集群回滚了
Git 仍然写着新版本
Argo CD 下一次同步可能又把新版本同步回来
```

所以 GitOps 回滚要让 Git 也回滚。

## 4. Argo CD rollback

Argo CD CLI/UI 有 rollback 能力。

它适合紧急情况，但使用后要把 Git 状态修正回来。

原则：

```text
紧急操作可以先救火。
救火后必须回写 Git。
```

## 5. Helm rollback 和 GitOps rollback

Helm rollback：

```bash
helm rollback go-cicd-lab 2
```

GitOps rollback：

```text
git revert <deploy commit>
git push
Argo CD sync
```

如果 Argo CD 管 Helm release，直接手动 Helm rollback 可能造成 drift。

GitOps 环境中不要绕过 Argo CD 长期操作。

## 6. 环境晋级

推荐：

```text
staging 验证 sha-8f3a2c1
-> 创建 release tag v0.1.0
-> production PR 把 image.tag 改为 v0.1.0
-> merge
-> Argo CD sync production
```

不要让 production 自动追踪 staging 的每个 sha。

## 7. 发布记录

部署仓库 commit 应该清晰：

```text
deploy: staging sha-8f3a2c1
deploy: production v0.1.0
rollback: production v0.1.0
```

PR 描述应包含：

- 镜像 tag。
- 应用 commit。
- 变更摘要。
- 数据库迁移说明。
- 回滚方式。

## 8. 数据库迁移仍然特殊

Git revert image tag 不会回滚数据库。

如果 release 包含破坏性迁移，回滚旧镜像可能失败。

所以仍然需要第 5、6 阶段讲过的：

```text
expand/contract
兼容迁移
备份
补偿脚本
```

## 9. 小练习

1. staging values 改到新 image tag。
2. 合并并同步。
3. 再 revert 这个 commit。
4. 观察 Argo CD 回到旧版本。
5. 在 UI 中查看 history 和 diff。

## 10. 本节小结

你现在应该理解：

- GitOps 回滚首选 Git revert。
- Argo CD/Helm 手动 rollback 后要修正 Git。
- production 发布适合通过 PR 晋级。
- 数据库迁移不会因为 Git revert 自动恢复。

