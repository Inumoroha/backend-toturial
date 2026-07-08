# 02：环境、配置与密钥

## 1. 本节目标

部署之前，先把环境、配置和密钥理清楚。

一次部署可以描述成：

```text
把某个镜像，带着某组配置和密钥，运行到某个环境。
```

## 2. 环境划分

本阶段建议至少两个环境：

```text
staging
production
```

staging：

- main 合并后自动部署。
- 用于部署验证和冒烟测试。
- 可以使用测试数据库。
- 可以自动回滚或手动修复。

production：

- 真实用户环境。
- 需要更严格权限。
- 推荐使用 tag 发布。
- 需要审批、记录和回滚方案。

## 3. 配置和密钥的区别

配置：

- `APP_ENV=staging`
- `HTTP_ADDR=:8080`
- `LOG_LEVEL=info`
- `PUBLIC_BASE_URL=https://staging.example.com`

密钥：

- `DATABASE_URL=postgres://user:password@host/db`
- `REDIS_PASSWORD=...`
- `JWT_SECRET=...`
- `GHCR_TOKEN=...`
- `SSH_PRIVATE_KEY=...`

配置可以不敏感，密钥必须保护。

## 4. 不要把环境写进镜像

错误做法：

```dockerfile
ENV DATABASE_URL=postgres://prod-user:prod-password@db/prod
```

正确思路：

```text
同一个镜像可以部署到 staging 或 production。
环境差异通过运行时配置和密钥注入。
```

镜像应该是环境无关的。

## 5. Compose 中的配置方式

推荐拆分：

```text
.env          # Compose 变量，例如 APP_IMAGE，不提交真实文件
app.env       # 应用环境变量，不提交真实文件
app.env.example
compose.yaml
```

`compose.yaml`：

```yaml
services:
  api:
    image: ${APP_IMAGE}
    env_file:
      - ./app.env
```

`.env`：

```text
APP_IMAGE=ghcr.io/your-name/go-cicd-lab:sha-abc123
```

`app.env`：

```text
APP_ENV=staging
HTTP_ADDR=:8080
DATABASE_URL=postgres://...
```

真实 `.env` 和 `app.env` 不提交。

## 6. GitHub Actions Environments

GitHub Actions environments 可以表示部署目标：

```text
staging
production
```

它们可以拥有：

- environment secrets。
- environment variables。
- required reviewers。
- branch restrictions。
- deployment history。

当 job 引用 environment 时：

```yaml
environment:
  name: production
```

如果 production 配置了 required reviewers，job 会等待审批。

## 7. Environment Secrets

推荐按环境存储：

staging secrets：

```text
STAGING_HOST
STAGING_USER
STAGING_SSH_KEY
STAGING_APP_DIR
```

production secrets：

```text
PRODUCTION_HOST
PRODUCTION_USER
PRODUCTION_SSH_KEY
PRODUCTION_APP_DIR
```

也可以在对应 environment 中使用统一名字：

```text
DEPLOY_HOST
DEPLOY_USER
DEPLOY_SSH_KEY
APP_DIR
```

然后不同 environment 放不同值。

## 8. 密钥最小权限

部署用 SSH key 应该：

- 只允许登录部署用户。
- 不使用 root。
- 只部署指定目录。
- 不和个人 SSH key 混用。
- 可随时轮换。

镜像仓库 token 应该：

- staging 只需要 pull 权限。
- production 只需要 pull 权限。
- CI 推镜像才需要 push 权限。

## 9. 配置漂移

配置漂移是指环境实际配置和文档或仓库描述不一致。

例如：

- staging 服务器手动改过 `.env`，没人记录。
- production 使用旧 compose 文件。
- 数据库地址和文档不一致。

减少漂移的方法：

- 配置模板提交到 Git。
- 真实密钥保存在 secret store。
- 部署脚本输出版本和配置摘要。
- 重要变更走 PR。

## 10. 小练习

为你的项目填写：

| 项目 | staging | production |
| --- | --- | --- |
| host | | |
| app dir | | |
| image registry | | |
| deploy user | | |
| required approval | | |
| app env file | | |
| secrets location | | |
| rollback version file | | |

## 11. 本节小结

你现在应该理解：

- 镜像应该环境无关。
- 环境差异通过运行时配置和密钥注入。
- GitHub environments 可以控制审批和环境密钥。
- production secret 不应该给 PR 或 staging job 使用。
- 配置漂移会让部署变得不可控。

