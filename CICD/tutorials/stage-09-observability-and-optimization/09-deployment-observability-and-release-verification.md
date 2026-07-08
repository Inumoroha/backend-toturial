# 09：部署可观测性与发布后验证

## 1. 本节目标

CI 通过不代表发布成功。

部署成功也不代表用户可用。

这一节学习：

- 如何记录部署事件。
- 如何关联 commit、workflow、image digest 和环境。
- 如何使用 Argo CD/Kubernetes 查看发布状态。
- 如何设计 smoke test。
- 如何写发布证据。

## 2. 发布证据链

每次发布至少能回答：

```text
谁触发的？
哪个 commit？
哪个 workflow run？
哪个镜像 digest？
部署到哪个环境？
何时开始和结束？
是否通过 smoke test？
如果失败，如何回滚？
```

建议记录：

```text
Git commit SHA
GitHub run URL
image tag
image digest
SBOM/provenance/signature
deploy repo commit
Argo CD application revision
Kubernetes rollout result
smoke test result
```

## 3. 在 Kubernetes 资源上加标签和注解

Helm values 示例：

```yaml
image:
  repository: ghcr.io/your-org/go-cicd-lab
  tag: ""
  digest: sha256:xxxx

deployment:
  annotations:
    gitSha: "abc123"
    workflowRun: "https://github.com/your-org/go-cicd-lab/actions/runs/123"
    imageDigest: "sha256:xxxx"
```

Deployment 模板中：

```yaml
metadata:
  annotations:
    app.example.com/git-sha: {{ .Values.deployment.annotations.gitSha | quote }}
    app.example.com/workflow-run: {{ .Values.deployment.annotations.workflowRun | quote }}
    app.example.com/image-digest: {{ .Values.deployment.annotations.imageDigest | quote }}
```

这样排障时可以从集群反查发布来源。

## 4. kubectl 查看发布状态

查看 rollout：

```bash
kubectl rollout status deployment/go-cicd-lab -n go-cicd-lab-staging
```

查看历史：

```bash
kubectl rollout history deployment/go-cicd-lab -n go-cicd-lab-staging
```

查看事件：

```bash
kubectl get events -n go-cicd-lab-staging --sort-by=.lastTimestamp
```

查看 Pod：

```bash
kubectl get pods -n go-cicd-lab-staging -l app=go-cicd-lab
kubectl describe pod <pod-name> -n go-cicd-lab-staging
```

## 5. Argo CD 查看发布状态

常用命令：

```bash
argocd app get go-cicd-lab-staging
argocd app history go-cicd-lab-staging
argocd app diff go-cicd-lab-staging
argocd app sync go-cicd-lab-staging
```

重点看：

```text
Sync Status：是否与 Git 一致。
Health Status：资源是否健康。
Revision：同步到哪个 Git commit。
History：谁在什么时候同步。
Conditions：是否有错误。
```

注意：

```text
Synced 不等于业务可用。
Healthy 也不等于所有业务路径正常。
还需要 smoke test 和运行时指标。
```

## 6. Smoke test 设计

smoke test 是发布后最小可用验证。

不要一开始写成完整 e2e。

建议包括：

- `/healthz` 返回 200。
- `/readyz` 返回 200。
- 关键 API 能完成一次最小业务请求。
- 数据库连接正常。
- 版本接口返回当前 commit 或版本。

示例：

```bash
BASE_URL="https://staging.example.com"

curl -fsS "$BASE_URL/healthz"
curl -fsS "$BASE_URL/readyz"
curl -fsS "$BASE_URL/version"
```

在 workflow 中：

```yaml
- name: Smoke test
  run: |
    curl -fsS "$BASE_URL/healthz"
    curl -fsS "$BASE_URL/readyz"
```

## 7. 发布后观察窗口

production 发布后，不要只看 workflow 成功。

建议观察：

```text
5 到 15 分钟内错误率。
p95/p99 延迟。
Pod 重启。
CPU/内存。
关键业务指标。
告警状态。
```

如果业务风险高，可以设置：

```text
发布后观察窗口未通过，不继续下一批发布。
```

这也是 canary/blue-green 发布的基础。

## 8. 发布 Summary

在 deployment workflow 中输出：

```yaml
- name: Deployment summary
  run: |
    {
      echo "## Deployment Summary"
      echo ""
      echo "| Item | Value |"
      echo "| --- | --- |"
      echo "| Environment | production |"
      echo "| Commit | \`${GITHUB_SHA}\` |"
      echo "| Image | \`${IMAGE}@${DIGEST}\` |"
      echo "| Argo CD App | go-cicd-lab-production |"
      echo "| Smoke Test | passed |"
    } >> "$GITHUB_STEP_SUMMARY"
```

## 9. 回滚信号

提前定义什么情况下回滚：

- smoke test 失败。
- 5xx 错误率超过阈值。
- p95 延迟超过阈值。
- Pod 大量 CrashLoopBackOff。
- 关键业务指标明显下降。
- 依赖服务异常且与发布相关。

没有定义回滚信号时，发布事故中容易争论：

```text
到底要不要回滚？
```

## 10. 小练习

完成：

1. 在 Helm values 中记录 commit、workflow run、image digest。
2. 发布后执行 `kubectl rollout status`。
3. 用 `argocd app history` 查看同步历史。
4. 写一个最小 smoke test 脚本。
5. 在 workflow summary 中输出发布证据。
6. 定义 3 个 production 回滚信号。

## 11. 本节小结

你现在应该理解：

- 发布必须留下 commit、run、digest、环境和验证结果。
- Kubernetes/Argo CD 状态只能说明部署层面，不等于业务可用。
- smoke test 是发布后最小可用验证。
- production 发布后需要观察窗口。
- 回滚信号要提前定义。
