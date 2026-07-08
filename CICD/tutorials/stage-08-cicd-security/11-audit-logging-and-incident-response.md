# 11：审计日志与事件响应

## 1. 本节目标

安全不是只做预防。

当 token 泄露、镜像被污染、workflow 被篡改、production 异常发布时，你需要快速回答：

```text
发生了什么？
影响范围多大？
现在如何止血？
如何恢复？
如何防止再次发生？
```

这一节学习：

- CI/CD 需要哪些审计线索。
- 如何设计事件响应流程。
- secret 泄露如何处理。
- 可疑镜像如何处理。
- 事后复盘怎么写。

## 2. 要保留哪些审计线索

至少关注这些系统：

```text
GitHub/GitLab audit log
workflow run log
artifact 下载记录
package/registry push 记录
container registry pull/push 记录
Argo CD application history
Kubernetes audit log
Kubernetes event
Ingress/API gateway log
云 IAM 操作日志
Secret Manager 访问日志
```

如果你暂时没有企业级审计能力，也要先养成习惯：

- 每次生产发布都有 PR。
- 每次镜像都有 digest。
- 每次 workflow 运行有 run id。
- 每次部署有 Argo CD history。
- 每次回滚有 Git commit。

## 3. 事件响应基本流程

可以按 6 步走：

```text
1. Triage：确认告警是否真实。
2. Contain：止血，阻止继续扩散。
3. Eradicate：移除恶意变更、吊销凭证、删除污染制品。
4. Recover：恢复服务和可信发布链路。
5. Verify：确认没有残留风险。
6. Postmortem：复盘并改进控制点。
```

不要一上来就只删除 Git 历史。

对于 secret 泄露，优先级是：

```text
撤销/轮换 secret > 查影响范围 > 清理历史 > 加固流程
```

## 4. 场景一：secret 泄露

可能来源：

- secret 被提交到 Git。
- workflow 日志打印了 secret。
- artifact 中包含 `.env`。
- Docker 镜像层包含 token。
- 个人 PAT 被复制到 CI。

响应步骤：

```text
1. 立即撤销或轮换 secret。
2. 暂停相关 workflow 或降低权限。
3. 检查 secret 使用系统的访问日志。
4. 查找泄露范围：Git 历史、日志、artifact、镜像层。
5. 重新发布不含 secret 的制品。
6. 清理历史和缓存。
7. 增加 secret scanning 和 review 规则。
```

检查 Git 历史可以使用：

```bash
git log --all --full-history -- path/to/leaked-file
```

但再次强调：

```text
删除历史不能让已经泄露的 secret 变安全。
真正让它失效的是撤销和轮换。
```

## 5. 场景二：workflow 被恶意修改

表现：

- workflow 权限突然变大。
- 新增 `pull_request_target`。
- 新增向外部地址上传 artifact。
- 新增打印环境变量。
- 第三方 action 换成陌生来源。

响应步骤：

```text
1. 立刻禁用可疑 workflow 或 revert。
2. 检查该 workflow 最近所有 run。
3. 检查是否访问过 secrets、packages、deploy key。
4. 轮换被访问过的凭证。
5. 检查发布过的镜像和 artifact。
6. 给 workflow 目录增加 CODEOWNERS。
7. 收紧 GITHUB_TOKEN permissions。
```

排查命令：

```bash
git log -- .github/workflows
git show <commit> -- .github/workflows/image.yml
```

## 6. 场景三：可疑镜像进入 registry

响应步骤：

```text
1. 找到镜像 tag 和 digest。
2. 确认是谁、哪个 workflow、哪个 commit 推送。
3. 检查镜像是否有签名和 provenance。
4. 如果不可信，阻止部署并删除或隔离 tag。
5. 如果已部署，立即回滚到可信 digest。
6. 检查 registry token 和 packages 权限。
```

查看本地镜像 digest：

```bash
docker buildx imagetools inspect ghcr.io/your-org/go-cicd-lab:tag
```

验证签名：

```bash
cosign verify \
  --certificate-identity-regexp "https://github.com/your-org/go-cicd-lab/.github/workflows/.*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/your-org/go-cicd-lab@sha256:xxxx
```

## 7. 场景四：production 异常发布

先看事实链：

```text
部署仓库 PR
合并 commit
Argo CD application history
镜像 digest
Kubernetes rollout history
业务指标和日志
```

优先恢复服务：

```bash
git revert <bad-deploy-commit>
```

然后让 Argo CD 同步回可信版本。

如果是 GitOps，避免只在集群中手动改回来，因为下一次 sync 可能再次回到错误状态。

## 8. 事件记录模板

建议创建：

```text
docs/incidents/YYYY-MM-DD-title.md
```

模板：

```markdown
# Incident: title

## Summary

## Timeline

- 10:00 Alert fired.
- 10:05 Deployment paused.
- 10:20 Token rotated.

## Impact

## Root Cause

## Detection

## Response

## What Went Well

## What Went Wrong

## Action Items

- [ ] Owner:
- [ ] Due date:
```

复盘不要只写“以后注意”。

好的 action item 应该能落到控制点：

```text
增加 branch protection。
增加 CODEOWNERS。
收紧 permissions。
改用 OIDC。
增加签名验证。
增加 secret scanning。
增加发布审批。
```

## 9. CI/CD 安全告警来源

可以逐步建立：

- secret scanning alert。
- Dependabot alert。
- code scanning alert。
- registry vulnerability alert。
- production deployment alert。
- Argo CD sync failure alert。
- Kubernetes admission denied alert。
- 云 IAM 异常登录或异常 AssumeRole alert。

告警不是越多越好。

更重要的是：

```text
每个告警有人负责。
每个高危告警有响应时限。
每个告警能关联到 run id、commit、digest 或 environment。
```

## 10. 小练习

完成：

1. 写一个 `docs/security-incident-runbook.md`。
2. 为 secret 泄露写响应流程。
3. 为可疑镜像写响应流程。
4. 为 production 异常发布写响应流程。
5. 记录你当前能查看哪些审计日志。
6. 记录你当前缺少哪些审计日志。

## 11. 本节小结

你现在应该理解：

- CI/CD 安全需要审计证据链。
- secret 泄露后先撤销轮换，不是先删历史。
- 可疑镜像要围绕 digest、签名和 provenance 排查。
- GitOps 回滚优先修改 Git。
- 复盘要输出可执行的加固项。

