# 02：Secret 管理与泄漏响应

## 1. 本节目标

Secret 是 CI/CD 中最危险的资产之一。

这一节学习：

- secret 分级。
- 存放位置。
- 环境隔离。
- 日志保护。
- 泄漏响应。

## 2. 什么是 secret

常见 secret：

- 数据库密码。
- JWT signing key。
- SSH private key。
- registry token。
- kubeconfig。
- 云厂商 access key。
- GitHub PAT。
- Argo CD repo credential。

判断标准：

```text
任何拿到它就能访问敏感系统或数据的内容，都是 secret。
```

## 3. Secret 分级

| 等级 | 示例 | 处理方式 |
| --- | --- | --- |
| 低 | 测试环境只读 token | 可放环境 secret，定期轮换 |
| 中 | staging deploy key | 环境隔离、最小权限 |
| 高 | production kubeconfig、云部署权限 | 审批、OIDC、短期凭证优先 |
| 极高 | 生产数据库管理员密码 | 尽量不进入 CI，使用 secret manager |

## 4. Secret 不该出现在哪里

不要出现在：

- Git 仓库。
- workflow YAML 明文。
- Dockerfile。
- 镜像层。
- CI 日志。
- PR 评论。
- issue。
- Wiki。
- 本地截图。

可以提交模板：

```text
.env.example
app.env.example
secret.example.yaml
```

模板里不写真值。

## 5. GitHub Secrets 放在哪里

优先级：

1. Environment secrets：部署到具体环境才可用。
2. Repository secrets：仓库级别。
3. Organization secrets：跨仓库共享。

production secret 推荐放 environment secrets，并加 required reviewers。

不要把 production secret 放到所有 workflow 都能读的宽泛位置。

## 6. 日志保护

不要打印：

```yaml
- run: env
- run: echo "$DATABASE_URL"
- run: echo "${{ toJson(secrets) }}"
```

如果脚本中生成了敏感值，使用 mask：

```bash
echo "::add-mask::$TOKEN"
```

注意：mask 不是万能的。最好的办法是不要输出 secret。

## 7. Secret 泄漏后怎么办

优先顺序：

1. 立即撤销或轮换 secret。
2. 检查使用范围。
3. 查看审计日志。
4. 删除或限制包含 secret 的 CI 日志。
5. 从代码中移除 secret。
6. 必要时清理 Git 历史。
7. 写事故记录和预防措施。

不要只做：

```text
删除 Git 里的 secret。
```

一旦推送到远程，就应该假设已经泄漏。

## 8. Secret scanning

GitHub Secret Scanning 可以扫描 Git 历史、PR、issue、discussion、wiki 和 gist 中的已知 secret 格式。

建议：

- 开启 secret scanning。
- 开启 push protection。
- 对内部格式配置 custom patterns。
- 收到 alert 后立即轮换。

## 9. Secret 轮换策略

建议记录：

| Secret | 所属环境 | 权限 | 创建时间 | 轮换周期 | Owner |
| --- | --- | --- | --- | --- | --- |
| GHCR_PULL_TOKEN | production | pull only | | 90 days | platform |
| DEPLOY_SSH_KEY | staging | deploy user | | 180 days | platform |

轮换要测试：

- 新 secret 可用。
- 旧 secret 已撤销。
- workflow 没有引用旧名字。

## 10. 小练习

1. 列出项目所有 secret。
2. 标记它们属于 staging 还是 production。
3. 判断是否最小权限。
4. 检查 workflow 是否打印敏感信息。
5. 写一份 secret 泄漏响应流程。

## 11. 本节小结

你现在应该理解：

- Secret 是 CI/CD 高价值攻击目标。
- production secret 应放在 protected environment 中。
- 泄漏后第一步是撤销和轮换。
- Secret scanning 是发现问题的工具，不是替代良好习惯。

