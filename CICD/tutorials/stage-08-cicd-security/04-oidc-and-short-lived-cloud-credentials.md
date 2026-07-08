# 04：OIDC 与短期云凭证

## 1. 本节目标

长期云密钥是 CI/CD 中最危险的 secret 之一。

OIDC 的目标：

```text
不在 GitHub Secrets 中保存长期云 Access Key。
让 workflow 用短期身份令牌换取短期云凭证。
```

## 2. OIDC 解决什么问题

传统方式：

```text
GitHub Secret 保存 AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY
workflow 使用它部署
```

风险：

- 长期有效。
- 泄漏后影响大。
- 轮换麻烦。
- 复制到多个仓库后难管理。

OIDC：

```text
GitHub workflow 请求 OIDC token
云平台验证 token claims
云平台发放短期凭证
```

## 3. OIDC 信任关系

云平台要配置：

```text
信任 GitHub OIDC provider。
只允许指定 org/repo/branch/environment 的 workflow 获取角色。
```

常见限制条件：

- repository。
- branch。
- tag。
- environment。
- workflow。
- actor，谨慎使用。

## 4. GitHub workflow 权限

必须配置：

```yaml
permissions:
  contents: read
  id-token: write
```

`id-token: write` 只允许请求 OIDC token。

真正云权限由云端 IAM role/policy 决定。

## 5. AWS 示例概念

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@<pinned-sha>
  with:
    role-to-assume: arn:aws:iam::<account-id>:role/github-actions-deploy
    aws-region: ap-east-1
```

生产建议 pin 到 commit SHA。

IAM trust policy 应限制：

```text
repo: your-org/go-cicd-lab
environment: production
branch/tag 条件
```

## 6. Azure/GCP 同理

Azure 通常使用：

```yaml
uses: azure/login@<pinned-sha>
```

GCP 通常使用：

```yaml
uses: google-github-actions/auth@<pinned-sha>
```

具体配置按云厂商官方文档执行。

本教程重点是模型，不绑定单一云。

## 7. Environment 保护

如果 OIDC 用于 production，建议 job 绑定 environment：

```yaml
environment:
  name: production
```

并在 environment 中配置：

- required reviewers。
- branch/tag restriction。
- wait timer，可选。

GitHub 官方也建议在 OIDC 策略与 environments 结合时使用环境保护规则。

## 8. OIDC 不适合什么

OIDC 不能替代所有 secret。

仍可能需要：

- 数据库密码。
- JWT secret。
- 第三方 API key。
- 私有 npm/go module token。

但云部署权限、云 registry 权限、Kubernetes 云身份应优先考虑 OIDC。

## 9. 小练习

设计一个 OIDC 策略：

| 问题 | 答案 |
| --- | --- |
| 云平台 | |
| 允许仓库 | |
| 允许环境 | |
| 允许 branch/tag | |
| 云角色权限 | |
| 是否需要 production approval | |

## 10. 本节小结

你现在应该理解：

- OIDC 用短期凭证替代长期云密钥。
- GitHub 需要 `id-token: write`。
- 云端 IAM trust policy 是安全核心。
- production OIDC 应结合 environment protection。
- OIDC 不能替代所有类型 secret。

