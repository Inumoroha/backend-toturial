# 阶段 5：CD 基础，从镜像到环境

> 本阶段目标：把第 4 阶段产出的镜像部署到真实运行环境，并建立 staging、production、健康检查、审批、回滚和部署排障的基础能力。

## 学习顺序

请按下面顺序学习：

1. [01-cd-model-and-deployment-flow.md](./01-cd-model-and-deployment-flow.md)
   - 理解 CD 的职责边界。
   - 学习从镜像发布到环境部署的完整流程。

2. [02-environments-config-and-secrets.md](./02-environments-config-and-secrets.md)
   - 学习 staging、production、配置和密钥。
   - 理解 GitHub Actions environments 和 environment secrets。

3. [03-docker-compose-runtime-stack.md](./03-docker-compose-runtime-stack.md)
   - 用 Docker Compose 定义 Go API、PostgreSQL、Redis。
   - 学习 `compose.yaml`、`.env`、`env_file`、healthcheck 和 volumes。

4. [04-single-vm-preparation.md](./04-single-vm-preparation.md)
   - 准备一台 Linux VM 作为 CD 入门部署目标。
   - 安装 Docker、配置目录、登录镜像仓库、创建最小权限用户。

5. [05-deploy-script-over-ssh.md](./05-deploy-script-over-ssh.md)
   - 编写 `deploy.sh`。
   - 通过 SSH 拉取镜像、更新 Compose、健康检查、记录版本。

6. [06-github-actions-deployment-workflow.md](./06-github-actions-deployment-workflow.md)
   - 编写 GitHub Actions 部署 workflow。
   - main 自动部署 staging，production 通过 environment 审批。

7. [07-smoke-tests-and-deployment-verification.md](./07-smoke-tests-and-deployment-verification.md)
   - 学习健康检查、ready 检查、冒烟测试。
   - 部署后用自动验证决定成功或失败。

8. [08-rollback-and-release-strategy.md](./08-rollback-and-release-strategy.md)
   - 学习镜像回滚、配置回滚和数据库迁移风险。
   - 设计 release、rollback、hotfix 的基本流程。

9. [09-database-migrations-in-cd.md](./09-database-migrations-in-cd.md)
   - 学习数据库迁移在 CD 中的位置。
   - 理解 expand/contract、向后兼容和破坏性变更。

10. [10-practice-deploy-go-cicd-lab.md](./10-practice-deploy-go-cicd-lab.md)
    - 为 `go-cicd-lab` 落地 Compose、部署脚本、GitHub Actions 部署 workflow。
    - 输出 `docs/deployment.md`。

11. [11-troubleshooting-and-operations.md](./11-troubleshooting-and-operations.md)
    - 学习部署失败排查、容器日志、Compose 状态、SSH 和 registry 问题。

12. [12-review-checklist-and-quiz.md](./12-review-checklist-and-quiz.md)
    - 用清单和自测题检查是否可以进入第 6 阶段。

## 本阶段建议时间

- 快速学习：4 到 7 天。
- 扎实学习：2 周。
- 推荐方式：先在本地 Docker Compose 跑通，再部署到一台测试 VM，最后接入 GitHub Actions。

## 本阶段你要准备什么

- 已完成第 4 阶段：能构建并推送镜像。
- 一台 Linux VM 或云服务器，作为 staging 练习环境。
- 一个镜像仓库，例如 GHCR。
- GitHub Actions 基础能力。

本地检查：

```bash
docker version
docker compose version
ssh -V
```

## 学完后你应该能做到

- 用 Docker Compose 定义 Go 服务运行栈。
- 在单台 VM 上运行和更新容器服务。
- 用 SSH 从 GitHub Actions 触发部署。
- 使用 GitHub environments 区分 staging 和 production。
- 为 production 增加人工审批。
- 部署后自动执行健康检查和冒烟测试。
- 失败时回滚到上一个镜像版本。
- 判断部署失败是镜像、配置、网络、密钥、数据库还是服务自身问题。

## 推荐官方资料

- GitHub Actions Deployments：<https://docs.github.com/actions/deployment/about-deployments/deploying-with-github-actions>
- GitHub Actions Environments：<https://docs.github.com/actions/deployment/targeting-different-environments/using-environments-for-deployment>
- GitHub Actions Secrets：<https://docs.github.com/actions/security-guides/encrypted-secrets>
- Docker Compose：<https://docs.docker.com/compose/>
- Docker Compose in production：<https://docs.docker.com/compose/how-tos/production/>
- Docker Compose CLI reference：<https://docs.docker.com/reference/cli/docker/compose/>

