# 13：CI/CD 安全审查清单

## 1. 本节目标

安全清单的作用不是增加流程负担，而是让你在关键变更前不漏掉高风险点。

这一节提供一组可以直接复制到 PR 模板、发布模板或团队规范中的清单。

## 2. 新增或修改 Workflow

- [ ] workflow 顶层设置了最小 `permissions`。
- [ ] job 级权限只在必要 job 放大。
- [ ] PR workflow 不读取部署 secret。
- [ ] 不在 fork PR 中执行高权限操作。
- [ ] 没有不必要的 `pull_request_target`。
- [ ] 第三方 actions 来自可信来源。
- [ ] 关键 workflow 的 actions 已 pin 到 tag 或 commit SHA。
- [ ] 不可信输入没有直接拼接进 shell 命令。
- [ ] artifact 不会从低信任 job 流向高权限执行步骤。
- [ ] workflow 文件由 CODEOWNERS 审批。

## 3. 新增 Secret

- [ ] secret 的用途明确。
- [ ] secret 有 owner。
- [ ] secret 有轮换周期。
- [ ] secret 权限最小化。
- [ ] secret 只放在需要的 repository/environment。
- [ ] production secret 受 environment 审批保护。
- [ ] secret 不会打印到日志。
- [ ] secret 不会写入 artifact。
- [ ] secret 不会写入容器镜像层。
- [ ] 有泄露后的撤销和轮换流程。

## 4. 使用 OIDC 或云凭证

- [ ] 优先使用 OIDC 短期凭证。
- [ ] 云角色信任策略限制 repository。
- [ ] 云角色信任策略限制 branch/tag/environment。
- [ ] 云角色权限只允许必要操作。
- [ ] 不使用长期云 AK/SK 进行常规部署。
- [ ] OIDC workflow 设置了 `id-token: write`。
- [ ] 只有需要云访问的 job 才拥有相关权限。

## 5. 新增第三方 Action

- [ ] action 维护者可信。
- [ ] action 仓库活跃维护。
- [ ] action 权限需求合理。
- [ ] action 不要求不必要的 secret。
- [ ] action 不上传未知数据到外部服务。
- [ ] 关键 workflow 已 pin 到 commit SHA。
- [ ] 有更新机制，例如 Dependabot。

## 6. Go 依赖变更

- [ ] 新依赖确实必要。
- [ ] 依赖来源可信。
- [ ] `go.mod` 变化符合预期。
- [ ] `go.sum` 没有大量无关变化。
- [ ] `go mod tidy` 已执行。
- [ ] `go test ./...` 通过。
- [ ] `govulncheck ./...` 通过或风险已记录。
- [ ] Dependabot/安全扫描结果已查看。

## 7. 镜像构建变更

- [ ] Dockerfile 使用多阶段构建。
- [ ] 运行镜像不包含 Go 编译器。
- [ ] 容器以非 root 用户运行。
- [ ] 没有把 secret 写入镜像。
- [ ] `.dockerignore` 排除了无关文件。
- [ ] 基础镜像来源可信。
- [ ] 生产镜像考虑使用 digest。
- [ ] 镜像扫描已通过。
- [ ] SBOM 已生成。
- [ ] provenance/attestation 已生成。
- [ ] 镜像已签名。

## 8. 发布到 Staging

- [ ] 镜像来自 main 或受信任分支。
- [ ] 镜像扫描通过。
- [ ] 部署使用明确 tag 或 digest。
- [ ] staging secret 与 production secret 隔离。
- [ ] staging 部署失败不会影响 production。
- [ ] smoke test 已执行。
- [ ] 失败时有回滚方式。

## 9. 发布到 Production

- [ ] production 变更通过 PR。
- [ ] CODEOWNERS 已审批。
- [ ] CI 必要检查全部通过。
- [ ] 镜像使用 digest。
- [ ] 镜像签名验证通过。
- [ ] provenance/attestation 可查询。
- [ ] SBOM 可查询。
- [ ] values diff 已人工确认。
- [ ] 没有意外 RBAC 变更。
- [ ] 没有意外 Ingress/TLS 变更。
- [ ] 有回滚 commit 或旧 digest。
- [ ] 发布窗口和影响范围已确认。

## 10. GitOps 变更

- [ ] 部署仓库分支受保护。
- [ ] production 目录需要 owner review。
- [ ] Argo CD Application 属于正确 AppProject。
- [ ] AppProject 限制 sourceRepos。
- [ ] AppProject 限制 destination namespace。
- [ ] Argo CD RBAC 没有给普通用户 admin。
- [ ] Secret 没有明文进入 Git。
- [ ] 变更能通过 Git revert 回滚。

## 11. Kubernetes 权限变更

- [ ] 没有默认授予 `cluster-admin`。
- [ ] Role/ClusterRole 只包含必要 resources。
- [ ] verbs 没有过度使用 `*`。
- [ ] ServiceAccount 只用于指定工作负载。
- [ ] production 和 staging 权限隔离。
- [ ] 新增 admission 或 policy 变更经过测试。

## 12. Emergency Access

紧急权限也要有边界：

- [ ] 有明确申请人和审批人。
- [ ] 有有效期。
- [ ] 有操作记录。
- [ ] 事后权限会撤销。
- [ ] 紧急修复会补回 Git。
- [ ] 事后有复盘。

不要让“紧急通道”变成长期后门。

## 13. 发布证据链

每次 production 发布建议记录：

```text
应用名
环境
发布时间
发布人
审批 PR
应用仓库 commit
部署仓库 commit
workflow run URL
镜像 tag
镜像 digest
SBOM 位置
provenance/attestation 位置
签名验证结果
回滚方式
```

## 14. 小练习

把本文件中的清单复制一部分到：

```text
.github/pull_request_template.md
docs/release-checklist.md
docs/security-review-checklist.md
```

然后用它审查你上一条 image workflow。

重点回答：

1. 哪些权限可以删掉？
2. 哪些 action 应该 pin？
3. 哪些 secret 可以改成 OIDC？
4. 哪些 production 变更需要额外审批？

## 15. 本节小结

你现在有了一套可复用的 CI/CD 安全审查清单。

清单的价值在于：

- 降低遗漏。
- 统一 review 标准。
- 把安全知识变成日常动作。
- 让生产发布留下清晰证据链。

