# 阶段 8：CI/CD 安全

> 本阶段目标：把前面已经跑通的 CI/CD 链路进行系统加固，重点保护源码、密钥、制品、镜像仓库、部署权限和生产环境。

## 学习顺序

请按下面顺序学习：

1. [01-threat-model-and-security-map.md](./01-threat-model-and-security-map.md)
   - 建立 CI/CD 威胁模型。
   - 识别源码、runner、secret、artifact、registry、cluster、GitOps 仓库中的风险点。

2. [02-secrets-management-and-leak-response.md](./02-secrets-management-and-leak-response.md)
   - 学习 secret 分级、环境隔离、轮换和泄漏响应。
   - 理解为什么泄漏后必须先撤销和轮换，而不是只删 Git 历史。

3. [03-github-token-permissions-and-least-privilege.md](./03-github-token-permissions-and-least-privilege.md)
   - 学习 `GITHUB_TOKEN`、`permissions`、job 级权限。
   - 把 workflow 权限从默认宽松改成最小权限。

4. [04-oidc-and-short-lived-cloud-credentials.md](./04-oidc-and-short-lived-cloud-credentials.md)
   - 学习 OIDC。
   - 用短期云凭证替代长期云密钥。

5. [05-third-party-actions-and-workflow-injection.md](./05-third-party-actions-and-workflow-injection.md)
   - 学习第三方 Action 供应链风险。
   - 学习 pin 到 commit SHA、避免脚本注入、谨慎处理 PR 输入。

6. [06-go-dependency-and-source-security.md](./06-go-dependency-and-source-security.md)
   - 学习 Go 依赖安全、`govulncheck`、Dependabot、CodeQL/代码扫描。
   - 把依赖漏洞管理纳入日常 PR。

7. [07-container-image-security.md](./07-container-image-security.md)
   - 学习镜像扫描、基础镜像加固、非 root、digest、漏洞阻断规则。
   - 用 Trivy 或平台扫描检查镜像。

8. [08-sbom-provenance-and-slsa.md](./08-sbom-provenance-and-slsa.md)
   - 学习 SBOM、provenance、GitHub Artifact Attestations、SLSA。
   - 理解“这个制品从哪里来、怎么构建、能否验证”。

9. [09-image-signing-and-verification.md](./09-image-signing-and-verification.md)
   - 学习 Cosign keyless signing。
   - 理解签名、验证、准入策略的关系。

10. [10-kubernetes-gitops-and-deployment-security.md](./10-kubernetes-gitops-and-deployment-security.md)
    - 加固 Kubernetes、Helm、Argo CD 和部署仓库权限边界。
    - 处理 kubeconfig、RBAC、AppProject、production PR 保护。

11. [11-audit-logging-and-incident-response.md](./11-audit-logging-and-incident-response.md)
    - 学习审计、日志、异常响应。
    - 编写 CI/CD 事故处理流程。

12. [12-practice-harden-go-cicd-lab.md](./12-practice-harden-go-cicd-lab.md)
    - 为 `go-cicd-lab` 落地安全加固。
    - 输出 `docs/security.md` 和若干 workflow 改造建议。

13. [13-security-review-checklist.md](./13-security-review-checklist.md)
    - 安全审查清单。
    - 用于每次新增 workflow、secret、部署权限前检查。

14. [14-review-checklist-and-quiz.md](./14-review-checklist-and-quiz.md)
    - 自测题和阶段验收标准。

## 本阶段建议时间

- 快速学习：1 周。
- 扎实学习：2 到 3 周。
- 推荐方式：先做权限和 secret 加固，再做供应链安全，再做签名和 provenance。

## 本阶段你要准备什么

- 已完成第 3 到第 7 阶段中的主要流水线。
- 熟悉 GitHub Actions。
- 已有镜像构建和部署流程。
- 有一个测试仓库可操作安全设置。

## 学完后你应该能做到

- 识别 CI/CD 关键攻击面。
- 为 workflow 设置最小 `GITHUB_TOKEN` 权限。
- 使用 GitHub environments 保护 production secret。
- 用 OIDC 替代长期云密钥。
- 管理第三方 Actions 风险。
- 使用 Dependabot、govulncheck、镜像扫描和 SBOM。
- 生成 provenance/attestation。
- 使用 Cosign 签名和验证镜像。
- 设计 secret 泄漏响应流程。
- 解释 GitOps 环境下的安全边界。

## 推荐官方资料

- GitHub Actions secure use：<https://docs.github.com/en/actions/reference/security/secure-use>
- GitHub OIDC：<https://docs.github.com/en/actions/concepts/security/openid-connect>
- GitHub OIDC in cloud providers：<https://docs.github.com/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-cloud-providers>
- GitHub Artifact Attestations：<https://docs.github.com/actions/security-for-github-actions/using-artifact-attestations>
- GitHub Secret Scanning：<https://docs.github.com/code-security/secret-scanning/about-secret-scanning>
- GitHub Dependabot：<https://docs.github.com/code-security/dependabot>
- Go govulncheck：<https://go.dev/doc/tutorial/govulncheck>
- Trivy：<https://trivy.dev/>
- Sigstore Cosign：<https://docs.sigstore.dev/quickstart/quickstart-cosign/>
- SLSA：<https://slsa.dev/>

