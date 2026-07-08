# 02：GitOps 仓库结构

## 1. 本节目标

GitOps 的关键是 Git 仓库如何组织。

这一节设计：

- 应用仓库。
- 部署仓库。
- 环境目录。
- Helm values。
- image tag 更新位置。

## 2. 单仓库还是双仓库

### 单仓库

```text
go-cicd-lab/
  app code
  deploy/helm/go-cicd-lab
  environments/staging
  environments/production
```

优点：

- 简单。
- 一处管理。

缺点：

- 应用代码变更和环境发布混在一起。
- production 发布 PR 不够独立。
- 多服务时扩展性差。

### 双仓库

```text
go-cicd-lab/
  app code
  Dockerfile
  CI

go-cicd-lab-deploy/
  charts/
  environments/
```

优点：

- 应用构建和环境发布分离。
- production 发布可以通过部署仓库 PR 管理。
- 权限边界更清楚。

缺点：

- 多一个仓库。
- CI 需要更新部署仓库。

推荐学习 GitOps 时使用双仓库。

## 3. 推荐部署仓库结构

```text
go-cicd-lab-deploy/
  charts/
    go-cicd-lab/
      Chart.yaml
      values.yaml
      templates/
  environments/
    staging/
      values.yaml
      application.yaml
    production/
      values.yaml
      application.yaml
  README.md
```

说明：

- `charts/` 保存 Helm Chart。
- `environments/staging/values.yaml` 保存 staging 差异。
- `environments/production/values.yaml` 保存 production 差异。
- `application.yaml` 保存 Argo CD Application。

## 4. values 文件怎么写

staging：

```yaml
replicaCount: 1

image:
  repository: ghcr.io/your-name/go-cicd-lab
  tag: sha-8f3a2c1

config:
  APP_ENV: staging
  LOG_LEVEL: debug
```

production：

```yaml
replicaCount: 3

image:
  repository: ghcr.io/your-name/go-cicd-lab
  tag: v0.1.0

config:
  APP_ENV: production
  LOG_LEVEL: info
```

GitOps 中 image tag 要写入 Git。

这和第 6 阶段 `helm upgrade --set image.tag=...` 不同。

## 5. 为什么 GitOps 中 tag 要落 Git

第 6 阶段：

```bash
helm upgrade --set image.tag=v0.1.0
```

集群状态变了，但 Git 不一定变。

GitOps 中：

```text
修改 environments/production/values.yaml
image.tag: v0.1.0
```

合并后 Argo CD 同步。

好处：

- Git 记录就是发布记录。
- 回滚可以 revert commit。
- 审计清晰。

## 6. 环境晋级

推荐：

```text
staging 使用 sha tag
production 使用 release tag
```

流程：

```text
main -> image sha-xxxx -> staging values 更新
tag v0.1.0 -> production values PR -> review -> merge
```

生产不要自动追踪 main。

## 7. Secret 不进部署仓库

部署仓库可以保存：

- Chart。
- values 中的非敏感配置。
- image tag。
- replicas。
- resources。
- ingress host。

不要保存：

- 数据库密码。
- JWT secret。
- 私钥。
- 云厂商 token。

Secret 应来自：

- External Secrets。
- Sealed Secrets。
- SOPS。
- 手动预创建 Secret。
- 云 Secret Manager。

第 8 阶段会更系统学习密钥。

## 8. 小练习

创建部署仓库目录草稿：

```text
go-cicd-lab-deploy/
  charts/go-cicd-lab/
  environments/staging/
  environments/production/
```

回答：

1. image tag 放在哪个文件？
2. production 是否自动更新？
3. secret 放在哪里？
4. Argo CD Application 放在哪里？

## 9. 本节小结

你现在应该理解：

- GitOps 推荐把应用代码和部署状态分离。
- 部署仓库保存环境期望状态。
- image tag 必须落 Git，才能让 Git 成为事实来源。
- production 发布适合通过部署仓库 PR。
- Secret 不应该明文进入部署仓库。

