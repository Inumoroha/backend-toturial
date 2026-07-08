# 阶段 10：毕业项目与面试准备

> 本阶段目标：把前 0 到 9 阶段学到的 Git、Go 质量门禁、CI、镜像、CD、Kubernetes、Helm、GitOps、安全、可观测和优化整合成一个完整毕业项目，并能在面试中清楚讲出来。

## 学习顺序

请按下面顺序学习：

1. [01-capstone-go-cicd-lab-overview.md](./01-capstone-go-cicd-lab-overview.md)
   - 理解毕业项目的目标、范围和最终交付物。
   - 把 `go-cicd-lab` 从练习项目升级为作品集项目。

2. [02-final-requirements-and-scoring-rubric.md](./02-final-requirements-and-scoring-rubric.md)
   - 明确最终验收要求。
   - 使用评分表检查项目完整度。

3. [03-application-repo-final-structure.md](./03-application-repo-final-structure.md)
   - 整理应用仓库结构。
   - 补齐 Go 服务、测试、Dockerfile、Makefile 和 workflow。

4. [04-deploy-repo-final-structure.md](./04-deploy-repo-final-structure.md)
   - 整理部署仓库结构。
   - 补齐 Helm chart、环境 values、Argo CD Application 和 AppProject。

5. [05-end-to-end-pipeline-implementation.md](./05-end-to-end-pipeline-implementation.md)
   - 串通从 PR 到 production 的完整流水线。
   - 明确每个 workflow 的触发条件、权限和产物。

6. [06-release-and-rollback-drill.md](./06-release-and-rollback-drill.md)
   - 练习一次正式发布。
   - 练习 GitOps 回滚、镜像回滚和事故记录。

7. [07-security-observability-and-optimization-review.md](./07-security-observability-and-optimization-review.md)
   - 复查安全、可观测性和优化成果。
   - 形成项目级 review 报告。

8. [08-documentation-system-and-demo-script.md](./08-documentation-system-and-demo-script.md)
   - 整理项目文档体系。
   - 写一份 10 到 15 分钟项目演示脚本。

9. [09-project-portfolio-readme.md](./09-project-portfolio-readme.md)
   - 写作品集级 README。
   - 让别人打开仓库就能理解你的工程能力。

10. [10-interview-storytelling-and-resume.md](./10-interview-storytelling-and-resume.md)
    - 把项目整理成简历描述和面试故事。
    - 学会用 STAR 方法讲项目。

11. [11-cicd-interview-questions-foundation.md](./11-cicd-interview-questions-foundation.md)
    - 复习 CI/CD 基础面试题。
    - 覆盖 Git、Go、CI、Docker、Kubernetes 和 CD。

12. [12-cicd-interview-questions-scenarios.md](./12-cicd-interview-questions-scenarios.md)
    - 练习场景题。
    - 覆盖失败排查、回滚、安全事故、性能优化和团队协作。

13. [13-mock-interview-and-gap-fix-plan.md](./13-mock-interview-and-gap-fix-plan.md)
    - 做一次模拟面试。
    - 根据结果生成补漏计划。

14. [14-final-review-checklist-and-next-steps.md](./14-final-review-checklist-and-next-steps.md)
    - 最终复习清单。
    - 给出后续成长路线。

## 本阶段建议时间

- 快速完成：1 周。
- 扎实完成：2 到 3 周。
- 推荐方式：先把项目跑通，再补文档和演示，最后集中准备面试表达。

## 本阶段你要准备什么

- 一个应用仓库：`go-cicd-lab`。
- 一个部署仓库：`go-cicd-lab-deploy`。
- GitHub Actions 或 GitLab CI 环境。
- 一个可用的 registry，例如 GHCR、Docker Hub 或 GitLab Container Registry。
- 一个部署目标：本地 kind/minikube、云 Kubernetes 或可演示的替代环境。
- 之前阶段形成的安全、可观测和优化文档。

## 学完后你应该能做到

- 独立讲清楚一条 Go 后端 CI/CD 全链路。
- 展示 PR 检查、镜像构建、扫描、签名、部署、回滚和观测。
- 给出真实的 workflow、Dockerfile、Helm chart、Argo CD Application。
- 解释为什么这样设计权限、缓存、分支保护和发布流程。
- 用数据说明流水线优化前后的变化。
- 回答常见 CI/CD 面试基础题和场景题。
- 把项目写进简历，并能在 10 到 15 分钟内演示。

## 最终交付物

```text
go-cicd-lab/
  README.md
  Makefile
  Dockerfile
  .dockerignore
  .github/workflows/
  docs/
  scripts/

go-cicd-lab-deploy/
  README.md
  charts/
  environments/
  applications/
  projects/
  docs/

portfolio/
  project-summary.md
  demo-script.md
  interview-notes.md
```

## 推荐复习资料

- 前 0 到 9 阶段的 `00-README.md`。
- 每个阶段的最终作业文件。
- 你的 workflow run 记录。
- 你的部署仓库 PR。
- 你的发布、回滚和优化报告。

