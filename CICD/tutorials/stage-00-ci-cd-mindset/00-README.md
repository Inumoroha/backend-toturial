# 阶段 0：建立 CI/CD 全局心智

> 本阶段目标：先不急着写流水线配置，而是理解 CI/CD 到底在解决什么问题，以及一条后端服务从提交代码到上线运行中间经历了什么。

## 学习顺序

按下面顺序阅读和练习：

1. [01-ci-cd-basic-concepts.md](./01-ci-cd-basic-concepts.md)
   - 理解 CI、Continuous Delivery、Continuous Deployment。
   - 搞清楚自动化流水线为什么重要。

2. [02-pipeline-building-blocks.md](./02-pipeline-building-blocks.md)
   - 理解流水线中的 trigger、job、step、runner、cache、artifact、secret、environment。
   - 先建立通用模型，后面学 GitHub Actions 或 GitLab CI 会轻松很多。

3. [03-from-commit-to-production.md](./03-from-commit-to-production.md)
   - 从一次代码提交开始，完整走一遍后端服务上线链路。
   - 明白测试、构建、镜像、部署、回滚分别在什么时候出现。

4. [04-environments-artifacts-and-secrets.md](./04-environments-artifacts-and-secrets.md)
   - 理解环境、制品和密钥。
   - 这是后续 CD、安全、Kubernetes 学习的基础。

5. [05-practice-draw-your-first-pipeline.md](./05-practice-draw-your-first-pipeline.md)
   - 动手画出你的第一条 Go 后端 CI/CD 流程。
   - 用一个小项目模拟真实团队交付过程。

6. [06-review-checklist-and-quiz.md](./06-review-checklist-and-quiz.md)
   - 用清单和题目检查是否真正理解第 0 阶段。
   - 如果答不出来，回到对应文件复习。

## 本阶段建议时间

- 快速学习：半天。
- 扎实学习：2 到 3 天。
- 推荐方式：每天读 1 到 2 篇，并完成里面的小练习。

## 本阶段不要求你掌握什么

暂时不要求你会写 GitHub Actions、GitLab CI、Dockerfile、Kubernetes YAML。

第 0 阶段只做一件事：建立清晰的地图。后面每学一个工具，你都知道它在整条链路里处于什么位置。

## 学完后你应该能说清楚

- CI 和 CD 分别解决什么问题。
- 一条流水线通常由哪些部分组成。
- cache 和 artifact 的区别。
- staging 和 production 的区别。
- 为什么生产发布必须可追溯、可回滚。
- 为什么密钥不能写在代码、日志或镜像里。

