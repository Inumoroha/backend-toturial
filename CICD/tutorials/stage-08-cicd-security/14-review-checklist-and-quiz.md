# 14：复习清单与自测题

## 1. 本节目标

用清单和问题确认你是否掌握第 8 阶段。

如果你能解释并实践 CI/CD 最小权限、secret 管理、依赖扫描、镜像扫描、SBOM、provenance、签名和 GitOps 部署安全，就可以进入第 9 阶段：可观测性与流水线优化。

## 2. 操作清单

### 威胁建模

- [ ] 我能画出 CI/CD 链路中的 source、runner、secret、artifact、registry、cluster、GitOps repo。
- [ ] 我能说明每个节点的主要风险。
- [ ] 我能区分 PR、main、release、production 的信任边界。

### Secret 与权限

- [ ] 我知道 secret 泄露后要先撤销和轮换。
- [ ] 我知道 production secret 应放在受保护 environment 中。
- [ ] 我能为 workflow 设置最小 `permissions`。
- [ ] 我知道 `GITHUB_TOKEN` 权限可以在 workflow 和 job 级别控制。
- [ ] 我能解释 OIDC 为什么优于长期云密钥。

### Actions 与注入风险

- [ ] 我知道第三方 action 是供应链依赖。
- [ ] 我知道生产关键 action 应 pin 到 commit SHA。
- [ ] 我知道 PR title/body/comment 都是不可信输入。
- [ ] 我知道 `pull_request_target` 的风险。
- [ ] 我不会执行低信任 artifact 中的脚本。

### Go 与源码安全

- [ ] 我能运行 `go mod tidy`。
- [ ] 我能运行 `go list -u -m all`。
- [ ] 我能运行 `govulncheck ./...`。
- [ ] 我能配置 Dependabot。
- [ ] 我知道 CodeQL/代码扫描和依赖扫描的区别。

### 镜像安全

- [ ] 我的 Dockerfile 使用多阶段构建。
- [ ] 生产镜像不使用 `golang` 作为运行镜像。
- [ ] 容器以非 root 运行。
- [ ] secret 不会进入镜像层。
- [ ] 我能用 Trivy 扫描镜像。
- [ ] 我能设置 HIGH/CRITICAL 漏洞门禁。

### SBOM、Provenance 与签名

- [ ] 我能解释 SBOM 是什么。
- [ ] 我能解释 provenance 是什么。
- [ ] 我能说明 SBOM 和 provenance 的区别。
- [ ] 我能生成镜像 SBOM。
- [ ] 我能生成 artifact attestation。
- [ ] 我能使用 Cosign keyless signing 签名镜像。
- [ ] 我能验证镜像签名的 identity 和 issuer。

### GitOps 与部署安全

- [ ] 我知道部署仓库是 production 期望状态来源。
- [ ] 我知道 production 目录需要 CODEOWNERS。
- [ ] 我能用 AppProject 限制 Argo CD 同步范围。
- [ ] 我知道真实 Secret 不应明文进入 Git。
- [ ] 我知道 production 发布要记录 digest、签名、SBOM、provenance。

### 审计与响应

- [ ] 我能列出 CI/CD 需要保留的审计线索。
- [ ] 我能写 secret 泄露响应步骤。
- [ ] 我能写可疑镜像响应步骤。
- [ ] 我知道 GitOps 回滚优先 `git revert`。
- [ ] 我能写一次安全事件复盘。

## 3. 自测题

### 题目 1：为什么不能只删除 Git 历史来处理 secret 泄露？

参考答案：

```text
因为 secret 一旦进入远端仓库、日志、缓存、artifact 或他人本地克隆，就可能已经被复制。
删除历史不能让 secret 失效。
正确优先级是先撤销或轮换 secret，再检查影响范围，最后清理历史并加固流程。
```

### 题目 2：为什么 workflow 顶层建议设置 `permissions: contents: read`？

参考答案：

```text
这是最小权限基线。
大多数 CI 检查只需要读取源码。
需要写 packages、attestations、security-events 或 id-token 的 job 再单独放大权限，可以降低被滥用后的影响范围。
```

### 题目 3：OIDC 相比长期云密钥有什么优势？

参考答案：

```text
OIDC 让 workflow 用短期身份向云厂商换取短期凭证。
仓库中不需要保存长期云 AK/SK。
云侧可以用 repository、branch、environment 等 claim 限制谁能换取凭证。
```

### 题目 4：为什么第三方 action 要 pin 到 commit SHA？

参考答案：

```text
分支和 tag 可能变化。
pin 到 commit SHA 可以让 workflow 执行内容更稳定、可审计、可复现。
代价是升级需要通过 Dependabot 或人工 review。
```

### 题目 5：SBOM 和 provenance 的区别是什么？

参考答案：

```text
SBOM 描述制品里包含哪些组件。
provenance 描述制品从哪个源码、哪个 workflow、哪个构建环境构建出来。
一个回答“里面有什么”，一个回答“怎么来的”。
```

### 题目 6：为什么镜像签名应该围绕 digest？

参考答案：

```text
tag 可以移动，digest 指向具体内容。
如果只围绕 tag 做验证，攻击者或误操作可以让同一个 tag 指向不同镜像。
签名和验证应绑定 image@sha256:digest。
```

### 题目 7：只验证镜像“有签名”够不够？

参考答案：

```text
不够。
还要验证签名 identity 是否来自可信仓库、可信 workflow、可信 branch/environment，以及 OIDC issuer 是否符合预期。
否则任何人给镜像签名都可能被误认为可信。
```

### 题目 8：GitOps 中为什么部署仓库需要强保护？

参考答案：

```text
部署仓库描述 production 期望状态。
如果攻击者能修改 production values，就可能让 Argo CD 部署恶意镜像或危险配置。
因此 production 目录需要分支保护、PR、CODEOWNERS、CI 校验和审计。
```

## 4. 情景题

### 情景 1

一个 PR 修改了 `.github/workflows/image.yml`，把权限从：

```yaml
permissions:
  contents: read
  packages: write
```

改成：

```yaml
permissions: write-all
```

你会怎么 review？

参考思路：

```text
要求说明每个新增权限的必要性。
拒绝使用 write-all 作为默认写法。
只给具体 job 增加必要权限，例如 packages: write、id-token: write、attestations: write。
检查是否同时修改了第三方 action、脚本上传、secret 访问等高风险点。
```

### 情景 2

有人在 production values 中把镜像从：

```text
ghcr.io/your-org/go-cicd-lab@sha256:trusted
```

改成：

```text
ghcr.io/random-user/go-cicd-lab:latest
```

你会怎么处理？

参考思路：

```text
拒绝合并。
production 应使用可信 registry、明确 digest，并验证签名、provenance 和 SBOM。
latest tag 不可用于 production。
如果是紧急修复，也要提供可信构建证据和回滚方案。
```

### 情景 3

Trivy 今天突然让旧镜像失败，昨天它还通过。

这是否说明 CI 坏了？

参考思路：

```text
不一定。
漏洞数据库更新后，旧镜像可能被发现新的漏洞。
应查看 CVE、严重等级、是否有修复版本、是否实际影响运行路径，然后按策略阻断、修复或记录风险接受。
```

### 情景 4

production 发生异常发布，但 Argo CD 显示是 Synced。

你会查什么？

参考思路：

```text
Synced 只表示集群与 Git 期望状态一致，不代表变更正确。
应检查部署仓库 PR、合并 commit、Argo CD history、镜像 digest、签名验证、workflow run、业务指标和日志。
如果 Git 中的期望状态错误，应 git revert 部署仓库 commit。
```

## 5. 第 8 阶段最终作业

为 `go-cicd-lab` 输出：

```text
.github/workflows/ci.yml
.github/workflows/codeql.yml
.github/workflows/image.yml
.github/dependabot.yml
.github/pull_request_template.md
docs/security.md
docs/security-incident-runbook.md
docs/release-checklist.md
```

要求：

- Go test、lint、`govulncheck` 能在 PR 运行。
- Dependabot 能更新 Go modules 和 GitHub Actions。
- 镜像构建后能扫描。
- `CRITICAL` 漏洞阻断发布。
- 镜像生成 SBOM。
- 镜像生成 provenance/attestation。
- 镜像使用 Cosign keyless signing。
- 能验证镜像签名 identity。
- production 发布使用部署仓库 PR。
- production 发布记录 digest、SBOM、provenance 和签名验证结果。
- secret 泄露有响应 runbook。

## 6. 进入下一阶段的标准

当你能做到下面几件事，就可以进入第 9 阶段：

1. 能画出 CI/CD 攻击面。
2. 能给 GitHub Actions 设置最小权限。
3. 能用 OIDC 替代长期云密钥。
4. 能处理第三方 action 和 workflow 注入风险。
5. 能用 `govulncheck` 和 Dependabot 管理 Go 依赖风险。
6. 能扫描镜像并制定漏洞门禁。
7. 能生成 SBOM 和 provenance。
8. 能签名并验证镜像。
9. 能保护 GitOps 部署仓库和 Argo CD 权限边界。
10. 能写 secret 泄露和可疑镜像响应流程。

第 9 阶段会学习 CI/CD 的可观测性与优化：流水线指标、构建耗时分析、缓存优化、失败率治理、部署指标、告警和持续改进。

