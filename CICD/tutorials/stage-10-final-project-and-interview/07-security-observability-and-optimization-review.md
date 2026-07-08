# 07：安全、可观测与优化复查

## 1. 本节目标

毕业项目前半段是跑通链路。

后半段要证明你能把链路经营好。

这一节复查三类进阶能力：

```text
安全
可观测
优化
```

## 2. 安全复查

检查应用仓库：

- [ ] workflow 默认 `contents: read`。
- [ ] 写权限只出现在需要的 job。
- [ ] PR workflow 不访问 production secret。
- [ ] 第三方 action 来源可信。
- [ ] 关键 action 有固定版本或 SHA 策略。
- [ ] Dependabot 已配置。
- [ ] `govulncheck` 已运行。
- [ ] 镜像扫描已运行。
- [ ] SBOM 已生成。
- [ ] provenance/attestation 已生成。
- [ ] 镜像已签名。

检查部署仓库：

- [ ] production 目录需要 PR。
- [ ] production 目录有 CODEOWNERS。
- [ ] Secret 不明文提交。
- [ ] AppProject 限制 sourceRepos。
- [ ] AppProject 限制 destination namespace。
- [ ] production values 使用 digest。

## 3. 安全文档输出

确认有：

```text
docs/security.md
docs/security-incident-runbook.md
docs/release-checklist.md
```

`docs/security.md` 至少说明：

- secret 管理。
- workflow 权限策略。
- 镜像扫描策略。
- SBOM/provenance/签名策略。
- production 发布保护。

## 4. 可观测复查

检查：

- [ ] workflow summary 输出关键信息。
- [ ] 测试报告 artifact 可下载。
- [ ] 镜像 digest 出现在 summary。
- [ ] 部署记录包含 commit 和 run URL。
- [ ] smoke test 可执行。
- [ ] Go 服务有 `/version`。
- [ ] Go 服务有 `/metrics`。
- [ ] 有 dashboard 设计或截图。
- [ ] 有告警规则或告警设计。
- [ ] 有发布观察窗口说明。

## 5. 可观测文档输出

确认有：

```text
docs/cicd-observability.md
docs/delivery-report-template.md
```

建议补充：

```markdown
## Release Verification

- Smoke test:
- Metrics:
- Logs:
- Rollback signal:

## Dashboards

- CI/CD dashboard:
- Production service dashboard:
```

## 6. 优化复查

检查：

- [ ] Go cache 已启用。
- [ ] Docker buildx cache 已启用。
- [ ] `.dockerignore` 合理。
- [ ] PR workflow 有 concurrency。
- [ ] production deploy 有串行 concurrency。
- [ ] test/lint/vuln 是否合理并行。
- [ ] 慢测试有分层策略。
- [ ] flaky test 有治理流程。
- [ ] artifact retention 合理。

## 7. 优化报告输出

确认有：

```text
docs/pipeline-optimization-report.md
```

至少包含：

```markdown
## Baseline

## Bottlenecks

## Changes

## Result

## Next Improvements
```

如果没有真实数据，也要写明：

```text
当前是本地或教学环境，真实团队中会从 GitHub API/Grafana/CI 数据平台采集。
```

## 8. 项目级 Review 报告

创建：

```text
docs/final-review.md
```

模板：

```markdown
# Final CI/CD Review

## Scope

## What Works

## Security Review

## Observability Review

## Optimization Review

## Known Gaps

## Next Steps
```

## 9. 面试表达重点

不要说：

```text
我用了 GitHub Actions、Docker、Kubernetes、Argo CD。
```

要说：

```text
我把 Go 服务的 PR 检查、镜像构建、安全扫描、GitOps 发布和发布后验证串成了一条链路。
其中 PR 阶段只做快速反馈，main 阶段构建可信镜像，production 通过部署仓库 PR 控制变更。
```

工具是手段，链路和权衡才是能力。

## 10. 小练习

完成：

1. 填写 `docs/final-review.md`。
2. 找出 3 个安全缺口。
3. 找出 3 个可观测缺口。
4. 找出 3 个优化空间。
5. 为每个缺口写一个后续计划。

## 11. 本节小结

你现在应该能从“能跑”走到“可审查”。

毕业项目的高级感来自：

- 权限边界清楚。
- 发布证据完整。
- 指标能说明问题。
- 文档能解释权衡。

