# 12：复习清单与自测题

## 1. 本节目标

用清单和问题确认你是否掌握第 5 阶段。

如果你能把镜像部署到 staging，并能验证和回滚，就可以进入第 6 阶段：Kubernetes 与 Helm。

## 2. 操作清单

### CD 基础

- [ ] 我能解释 CI 和 CD 的边界。
- [ ] 我知道部署成功不等于命令执行成功。
- [ ] 我能描述从 image 到 environment 的部署流程。
- [ ] 我知道 staging 自动、production 审批的价值。

### 环境与密钥

- [ ] 我能区分配置和密钥。
- [ ] 我知道镜像应该环境无关。
- [ ] 我会使用 GitHub Actions environments。
- [ ] 我知道 production secrets 不应该给 staging job 使用。
- [ ] 我知道 SSH key 和 registry token 都要最小权限。

### Docker Compose

- [ ] 我能写 `compose.yaml`。
- [ ] 我知道 `.env` 和 `env_file` 的区别。
- [ ] 我能使用 named volumes。
- [ ] 我能运行 `docker compose up -d`。
- [ ] 我能运行 `docker compose pull`。
- [ ] 我能查看 `docker compose ps` 和 `docker compose logs`。

### VM 部署

- [ ] 我能准备 deploy 用户。
- [ ] 我能配置 SSH key。
- [ ] 我能登录镜像仓库。
- [ ] 我能在 VM 上手动运行部署脚本。
- [ ] 我能记录当前镜像和上一个镜像。

### GitHub Actions 部署

- [ ] 我能创建 `deploy.yml`。
- [ ] 我能用 `workflow_dispatch` 手动部署。
- [ ] 我能 main 自动部署 staging。
- [ ] 我能 tag `v*` 触发 production。
- [ ] 我能给 production 加 required reviewers。
- [ ] 我知道 environment secrets 何时可访问。

### 验证和回滚

- [ ] 我有 `/healthz`。
- [ ] 我有 `/readyz`。
- [ ] 我理解 `/version` 的价值。
- [ ] 我能写冒烟测试。
- [ ] 我能回滚到指定镜像。
- [ ] 我知道数据库迁移可能让回滚变复杂。

## 3. 自测题

### 题目 1：为什么部署成功不能只看 `docker compose up -d` 是否成功？

参考答案：

```text
`docker compose up -d` 成功只说明容器启动命令执行完成。
服务可能随后崩溃，或者虽然进程存在但数据库不可达。
必须通过 health check、ready check、冒烟测试和日志确认服务真正可用。
```

### 题目 2：staging 和 production 的 CD 规则应该有什么区别？

参考答案：

```text
staging 可以在 main 合并后自动部署，用于快速验证。
production 面向真实用户，应该使用明确版本、环境审批、受保护密钥、发布记录和回滚方案。
```

### 题目 3：为什么同一个镜像应该能部署到不同环境？

参考答案：

```text
镜像应该代表代码和运行时制品，环境差异应该通过运行时配置和密钥注入。
这样同一个镜像可以在 staging 验证后再进入 production，减少“环境专属镜像”导致的不一致。
```

### 题目 4：`.env` 和 `app.env` 可以怎么分工？

参考答案：

```text
`.env` 可以给 Compose 做变量替换，例如 APP_IMAGE 和数据库容器初始化变量。
`app.env` 可以通过 env_file 注入应用容器，例如 APP_ENV、DATABASE_URL、REDIS_ADDR。
真实文件不应该提交，只提交 example 模板。
```

### 题目 5：production environment required reviewers 有什么价值？

参考答案：

```text
它让 production job 在执行前等待人工审批。
审批通过前，job 不会运行，也不能访问该 environment 的 secrets。
这能防止未经确认的生产发布。
```

### 题目 6：回滚为什么不只是重新部署旧镜像？

参考答案：

```text
如果事故只来自应用代码，部署旧镜像可能足够。
但如果同时改了配置、数据库 schema 或数据，旧镜像可能无法运行。
所以回滚要同时考虑镜像、配置、数据库和验证。
```

### 题目 7：为什么数据库删除字段是高风险迁移？

参考答案：

```text
旧版本代码可能还依赖该字段。
如果发布失败想回滚应用，旧代码可能因为字段不存在而无法运行。
删除字段通常应该放在确认没有旧代码依赖后的后续版本。
```

### 题目 8：部署失败时你会按什么顺序排查？

参考答案：

```text
先定位失败层级：GitHub Actions 触发和审批、SSH、registry pull、Docker Compose、容器日志、应用健康检查、数据库依赖、服务器资源。
然后进入具体日志，而不是直接猜原因。
```

## 4. 情景题

### 情景 1

main 合并后 staging 部署失败，日志显示 `pull access denied`。

你会检查什么？

参考思路：

```text
检查镜像名和 tag 是否存在。
检查服务器是否登录 registry。
检查镜像是否私有。
检查 token 是否有 pull 权限。
检查 image workflow 是否真的推送了该 tag。
```

### 情景 2

production 发布后 `/healthz` 成功，但 `/readyz` 失败。

你会怎么理解？

参考思路：

```text
进程还活着，但服务未准备好接流量。
通常要检查数据库、Redis、外部依赖、配置和迁移状态。
这类发布应该判定失败，不能只看 healthz。
```

### 情景 3

部署脚本失败后，`.env` 已经被改成新镜像，但服务没 ready。

你怎么处理？

参考思路：

```text
先查看 docker compose ps 和 api 日志。
如果确认新版本不可用，用旧镜像重新执行 deploy.sh。
后续可以增强脚本：失败时自动恢复 .env 中 APP_IMAGE，或增加 rollback 脚本。
```

## 5. 第 5 阶段最终作业

在 `go-cicd-lab` 中完成：

```text
deploy/compose/compose.yaml
deploy/compose/.env.example
deploy/compose/app.env.example
scripts/deploy.sh
scripts/smoke-test.sh
.github/actions/ssh-deploy/action.yml
.github/workflows/deploy.yml
docs/deployment.md
```

要求：

- VM 上能手动部署。
- GitHub Actions 能手动部署 staging。
- main 能自动部署 staging。
- tag `v*` 能触发 production 审批。
- 部署后自动检查 `/healthz` 和 `/readyz`。
- 部署记录包含当前镜像和上一个镜像。
- 能手动回滚到指定镜像。

## 6. 进入下一阶段的标准

当你能做到下面几件事，就可以进入第 6 阶段：

1. 能解释 CD 中环境、配置、密钥、镜像的关系。
2. 能用 Compose 在单机运行 Go 服务。
3. 能通过 SSH 和 GitHub Actions 部署镜像。
4. 能为 production 设置审批门禁。
5. 能通过健康检查判断部署成功。
6. 能回滚到上一个稳定镜像。

第 6 阶段会把这些概念迁移到 Kubernetes 和 Helm：Deployment、Service、ConfigMap、Secret、readiness/liveness、Helm values 和滚动更新。

