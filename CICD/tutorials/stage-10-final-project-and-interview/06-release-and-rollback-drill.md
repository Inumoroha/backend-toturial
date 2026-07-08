# 06：发布与回滚演练

## 1. 本节目标

没有演练过的回滚，事故发生时往往不好用。

这一节学习如何做一次完整发布演练和回滚演练。

你要输出：

```text
docs/release-drill.md
docs/rollback-drill.md
docs/incidents/demo-rollback.md
```

## 2. 发布演练准备

选择一个小变更：

```text
修改 /version 输出。
新增一个 harmless endpoint。
修改 README 中显示的版本号。
```

不要用大功能做第一次演练。

发布前确认：

- PR checks 全部通过。
- 镜像已构建。
- 镜像 digest 已记录。
- staging smoke test 通过。
- production PR 已准备。
- 回滚目标版本已明确。

## 3. 发布记录模板

创建：

```text
docs/release-drill.md
```

内容：

```markdown
# Release Drill

Date:
Environment:
Release owner:

## Change

- Application commit:
- Deploy repo commit:
- Image:
- Digest:
- Workflow run:

## Pre-checks

- [ ] CI passed.
- [ ] Image scan passed.
- [ ] Signature verified.
- [ ] Staging smoke test passed.
- [ ] Rollback target identified.

## Execution

- Start time:
- Argo CD sync:
- Kubernetes rollout:
- Smoke test:
- End time:

## Result

## Notes
```

## 4. 发布命令

查看当前版本：

```bash
curl -fsS "$BASE_URL/version"
```

查看 Argo CD：

```bash
argocd app get go-cicd-lab-production
argocd app history go-cicd-lab-production
```

同步：

```bash
argocd app sync go-cicd-lab-production
```

查看 rollout：

```bash
kubectl rollout status deployment/go-cicd-lab -n go-cicd-lab-production
```

smoke test：

```bash
BASE_URL="https://production.example.com" bash scripts/smoke-test.sh
```

## 5. 回滚方式一：Git revert

GitOps 首选：

```bash
git revert <bad-deploy-commit>
git push
```

然后：

```bash
argocd app sync go-cicd-lab-production
```

优点：

```text
Git 期望状态也回滚。
审计记录清晰。
下一次同步不会又回到错误版本。
```

## 6. 回滚方式二：修改 image digest

如果不方便 revert，可以创建 PR 把 production values 改回旧 digest。

示例：

```yaml
image:
  digest: "sha256:old-trusted-digest"
```

然后合并 PR，同步 Argo CD。

注意：

```text
不要只在集群里手动 kubectl set image，除非是紧急止血。
紧急手动修复后也要补回 Git。
```

## 7. 回滚演练模板

创建：

```text
docs/rollback-drill.md
```

内容：

```markdown
# Rollback Drill

Date:
Environment:
Owner:

## Bad Release

- Deploy commit:
- Image digest:
- Symptoms:

## Rollback Target

- Deploy commit:
- Image digest:

## Steps

1.
2.
3.

## Verification

- [ ] Argo CD synced.
- [ ] Rollout completed.
- [ ] Smoke test passed.
- [ ] Metrics returned to normal.

## Duration

- Detection:
- Decision:
- Recovery:
```

## 8. 事故记录演练

创建：

```text
docs/incidents/demo-rollback.md
```

内容：

```markdown
# Incident: Demo Rollback

## Summary

## Timeline

## Impact

## Root Cause

## Detection

## Response

## Recovery

## Follow-up Actions
```

即使是演练，也按真实事故格式记录。

## 9. 回滚决策信号

提前定义：

- smoke test 失败。
- 5xx 错误率超过阈值。
- p95 延迟超过阈值。
- Pod CrashLoopBackOff。
- 关键业务指标下降。
- 数据库迁移失败。

面试时你可以说：

```text
我不会等到所有人争论完才回滚。项目里提前定义了回滚信号和流程。
```

## 10. 小练习

完成一次：

1. 发布一个小变更到 staging。
2. 记录发布证据。
3. 创建 production PR。
4. 模拟 production 发布。
5. 使用 Git revert 回滚。
6. 记录回滚耗时和结果。

## 11. 本节小结

你现在应该理解：

- 发布和回滚都需要演练。
- GitOps 回滚优先修改 Git。
- 回滚目标版本要提前知道。
- 演练记录能成为面试中的强证据。
- 回滚流程越熟，事故中越不慌。

