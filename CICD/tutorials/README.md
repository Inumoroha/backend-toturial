# CI/CD 教程目录

> 每个阶段放在独立文件夹中，文件夹和 Markdown 文件都使用数字前缀表示学习顺序。

## 当前教程

1. [stage-00-ci-cd-mindset](./stage-00-ci-cd-mindset/)
   - 建立 CI/CD 全局心智。
   - 理解流水线组成、环境、制品、密钥、发布和回滚。
   - 完成第一条 Go 后端 CI/CD 流程设计。

2. [stage-01-git-branching-and-collaboration](./stage-01-git-branching-and-collaboration/)
   - 学习 Git 核心模型、日常命令和分支策略。
   - 理解 PR/MR、代码评审、tag、release 和仓库保护规则。
   - 为 Go 后端项目设计可触发 CI/CD 的协作流程。

3. [stage-02-go-quality-gates](./stage-02-go-quality-gates/)
   - 学习 Go 项目的格式化、测试、构建、lint 和安全检查。
   - 掌握 `go test`、race、coverage、`golangci-lint`、`govulncheck`。
   - 用 Makefile 建立可被 CI 复用的本地质量门禁。

4. [stage-03-first-ci-pipeline](./stage-03-first-ci-pipeline/)
   - 学习 GitHub Actions/GitLab CI 的 workflow、job、step 和 runner。
   - 把 Go 本地质量门禁搬进 PR 和 main 分支 CI。
   - 掌握缓存、artifact、matrix、required checks 和 CI 安全边界。

5. [stage-04-docker-artifacts-and-registry](./stage-04-docker-artifacts-and-registry/)
   - 学习 Dockerfile、多阶段构建、`.dockerignore` 和本地镜像运行。
   - 掌握镜像 tag、label、digest、registry、GHCR/GitLab/Docker Hub。
   - 在 CI 中构建、扫描并推送 Go 服务镜像。

6. [stage-05-cd-basics](./stage-05-cd-basics/)
   - 学习从镜像到环境的 CD 基础流程。
   - 使用 Docker Compose、SSH 和 GitHub Actions 部署 Go 服务到单台 VM。
   - 掌握 staging/production、审批、健康检查、冒烟测试、回滚和部署排障。

7. [stage-06-kubernetes-and-helm](./stage-06-kubernetes-and-helm/)
   - 学习 Kubernetes Pod、Deployment、Service、ConfigMap、Secret、Ingress、Job。
   - 使用 readiness/liveness/startup probes、resources、rollout 和 rollback 管理 Go 服务。
   - 用 Helm Chart、values 和 release 管理多环境部署，并在 CI 中校验和部署。

8. [stage-07-gitops-and-argocd](./stage-07-gitops-and-argocd/)
   - 学习 GitOps、期望状态、drift、pull-based CD 和 Argo CD。
   - 使用 Argo CD Application 从 Git 同步 Helm Chart 到 Kubernetes。
   - 掌握自动同步、prune、selfHeal、部署仓库 image tag 更新、回滚、AppProject 和 RBAC。

9. [stage-08-cicd-security](./stage-08-cicd-security/)
   - 学习 CI/CD 威胁建模、secret 管理、最小权限、OIDC 和第三方 Actions 安全。
   - 掌握 Go 依赖安全、镜像扫描、SBOM、provenance、SLSA 和 Cosign 签名验证。
   - 加固 Kubernetes/GitOps 部署边界，并建立审计、事件响应和发布安全清单。

10. [stage-09-observability-and-optimization](./stage-09-observability-and-optimization/)
    - 学习 CI/CD 可观测性、workflow 日志、指标采集、瓶颈分析和优化基线。
    - 掌握 Go/Docker 缓存、并行 job、matrix、concurrency、flaky test 治理和发布后验证。
    - 建立 Go 服务 metrics、dashboard、告警、DORA 指标、成本治理和交付报告。

11. [stage-10-final-project-and-interview](./stage-10-final-project-and-interview/)
    - 整合前 0 到 9 阶段内容，完成 `go-cicd-lab` 毕业项目。
    - 梳理应用仓库、部署仓库、端到端流水线、发布回滚、安全观测和优化报告。
    - 准备作品集 README、演示脚本、简历描述、基础题、场景题和模拟面试。

## 后续阶段规划

后续可以继续按这个结构扩展：

```text
tutorials/
  stage-00-ci-cd-mindset/
  stage-01-git-branching-and-collaboration/
  stage-02-go-quality-gates/
  stage-03-first-ci-pipeline/
  stage-04-docker-artifacts-and-registry/
  stage-05-cd-basics/
  stage-06-kubernetes-and-helm/
  stage-07-gitops-and-argocd/
  stage-08-cicd-security/
  stage-09-observability-and-optimization/
  stage-10-final-project-and-interview/
  stage-11-production-practice-and-platform-engineering/
```

建议每个阶段都保留一个 `00-README.md`，用来说明本阶段的学习顺序、目标和验收标准。
