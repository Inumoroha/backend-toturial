# 阶段 3：第一个 CI 流水线

> 本阶段目标：把第 2 阶段建立的 Go 本地质量门禁搬进 CI 平台，让每个 PR 和 main 分支提交都能自动执行检查。

## 学习顺序

请按下面顺序学习：

1. [01-ci-platform-and-workflow-model.md](./01-ci-platform-and-workflow-model.md)
   - 理解 CI 平台、runner、workflow、job、step。
   - 明确第 3 阶段只做 CI，不急着做部署。

2. [02-github-actions-first-workflow.md](./02-github-actions-first-workflow.md)
   - 创建第一条 GitHub Actions workflow。
   - 让 PR 和 main push 自动运行 Go 测试与构建。

3. [03-triggers-jobs-steps-and-status-checks.md](./03-triggers-jobs-steps-and-status-checks.md)
   - 深入理解 `on`、`jobs`、`steps`、`needs`、required checks。
   - 学会把 CI 结果接到 PR 合并门禁。

4. [04-integrate-makefile-and-go-quality-gates.md](./04-integrate-makefile-and-go-quality-gates.md)
   - 在 CI 中复用第 2 阶段的 Makefile。
   - 接入 fmt、vet、test、race、coverage、build、lint、vuln。

5. [05-cache-artifacts-matrix-and-reports.md](./05-cache-artifacts-matrix-and-reports.md)
   - 学习 Go 依赖缓存、覆盖率 artifact、matrix。
   - 让 CI 更快、更可追溯。

6. [06-secrets-permissions-and-pr-security.md](./06-secrets-permissions-and-pr-security.md)
   - 学习 `GITHUB_TOKEN`、`permissions`、secret 和 PR 安全。
   - 避免初学阶段就把生产权限暴露给 CI。

7. [07-gitlab-ci-equivalent.md](./07-gitlab-ci-equivalent.md)
   - 用 GitLab CI/CD 写同等能力的 Go CI。
   - 理解 GitHub Actions 和 GitLab CI 的概念映射。

8. [08-practice-build-ci-for-go-cicd-lab.md](./08-practice-build-ci-for-go-cicd-lab.md)
   - 为 `go-cicd-lab` 落地第一条完整 CI。
   - 输出 `.github/workflows/ci.yml` 和文档。

9. [09-troubleshooting-and-optimization.md](./09-troubleshooting-and-optimization.md)
   - 学习 CI 失败排查、日志阅读、耗时优化。
   - 建立“不靠猜，靠日志定位”的习惯。

10. [10-review-checklist-and-quiz.md](./10-review-checklist-and-quiz.md)
    - 用清单和自测题确认是否可以进入第 4 阶段。

## 本阶段建议时间

- 快速学习：2 到 3 天。
- 扎实学习：1 到 2 周。
- 推荐方式：先写最小 CI，再逐步增强。不要一开始把所有功能塞进一个复杂 YAML。

## 本阶段你要准备什么

- 一个 GitHub 或 GitLab 仓库。
- 已完成第 2 阶段的 Go 本地质量门禁。
- 项目根目录中最好已有：

```text
go.mod
go.sum
Makefile
.golangci.yml
```

## 学完后你应该能做到

- 创建 `.github/workflows/ci.yml`。
- 在 PR 和 main push 时自动运行 CI。
- 使用 `actions/checkout` 和 `actions/setup-go`。
- 在 CI 中执行 Go 测试、构建、lint 和漏洞检查。
- 上传覆盖率报告 artifact。
- 使用 matrix 测试多个 Go 版本。
- 为 main 分支配置 required status checks。
- 解释为什么 PR CI 不应该使用生产 secret。

## 推荐官方资料

- GitHub Actions 文档：<https://docs.github.com/actions>
- GitHub Actions Workflow 语法：<https://docs.github.com/actions/using-workflows/workflow-syntax-for-github-actions>
- GitHub Actions 构建和测试 Go：<https://docs.github.com/actions/automating-builds-and-tests/building-and-testing-go>
- GitHub Actions GITHUB_TOKEN：<https://docs.github.com/actions/reference/authentication-in-a-workflow>
- GitHub Actions workflow artifacts：<https://docs.github.com/actions/using-workflows/storing-workflow-data-as-artifacts>
- golangci-lint CI 安装：<https://golangci-lint.run/docs/welcome/install/ci/>
- govulncheck Action：<https://github.com/golang/govulncheck-action>
- GitLab CI/CD 快速开始：<https://docs.gitlab.com/ci/quick_start/>
- GitLab CI/CD YAML 参考：<https://docs.gitlab.com/ci/yaml/>

