# 10：面试表达与简历写法

## 1. 本节目标

项目做完后，要能讲。

这一节学习：

- 如何把项目写进简历。
- 如何用 STAR 方法讲项目。
- 如何回答“你负责了什么”。
- 如何讲技术权衡。

## 2. 简历项目描述

不要写：

```text
熟悉 GitHub Actions、Docker、Kubernetes、Argo CD。
```

这像关键词堆砌。

更好：

```text
设计并实现 Go 后端服务 CI/CD 示例项目，覆盖 PR 质量门禁、Docker 镜像构建、Trivy 扫描、SBOM/provenance、Cosign 签名、Helm + Argo CD GitOps 部署、production PR 发布、回滚演练和流水线优化报告。
```

如果有数据：

```text
通过 Go/Docker 缓存和 PR concurrency，将示例流水线 p50 耗时从 8 分钟优化到 4 分钟。
```

没有真实数据时不要编。

可以写：

```text
建立了可观测指标和优化报告模板，用于记录 CI p50/p95、失败率和构建瓶颈。
```

## 3. STAR 方法

STAR：

```text
Situation：背景。
Task：任务。
Action：行动。
Result：结果。
```

示例：

```text
S：我希望把一个 Go API 项目从手动构建部署升级成可复现的 CI/CD 链路。
T：目标是让 PR 自动检查、main 自动构建可信镜像，并通过 GitOps 部署到 staging/production。
A：我拆分了 ci、image、deploy workflows，引入 Go 测试、govulncheck、镜像扫描、SBOM、签名和部署仓库 PR。
R：最终形成了一条从代码提交到发布验证和回滚的完整链路，并补充了 runbook、优化报告和演示脚本。
```

## 4. 两分钟项目介绍

准备一个短版：

```text
我做了一个 Go 后端 CI/CD 毕业项目，业务上是一个简单 Todo API，但重点在工程链路。
应用仓库负责 Go 代码、测试、镜像构建和安全扫描；部署仓库负责 Helm values 和 Argo CD Application。
PR 阶段跑测试、lint 和漏洞检查，main 合并后构建镜像、生成 SBOM/provenance 并签名。
staging 可以自动更新，production 通过部署仓库 PR 发布，并通过 Argo CD 同步。
我还补了 smoke test、/metrics、发布和回滚 runbook，以及一份流水线优化报告。
```

背熟，但不要机械朗读。

## 5. 五分钟架构讲解

结构：

```text
1. 业务服务：Go API + PostgreSQL。
2. 应用仓库：测试、镜像、安全、制品。
3. 部署仓库：Helm、环境 values、GitOps。
4. 发布策略：staging 自动，production PR。
5. 可靠性：回滚、可观测、优化。
```

每一段都给一个具体例子。

## 6. 讲权衡

面试官喜欢听权衡。

准备 5 个：

### 权衡 1：一个 workflow 还是多个 workflow

```text
我把 PR 检查、镜像构建和部署更新拆开。
好处是权限更小、职责更清楚、失败更容易定位。
代价是 workflow 数量变多，需要用文档解释链路。
```

### 权衡 2：staging 自动，production PR

```text
staging 追求快速反馈，可以自动部署。
production 更重视审计和控制，所以通过部署仓库 PR、CODEOWNERS 和审批发布。
```

### 权衡 3：tag 和 digest

```text
tag 适合人读，但可能移动。
production 使用 digest 可以确保部署的是明确镜像内容。
```

### 权衡 4：缓存和可复现

```text
缓存能加速 CI，但不能让缓存绕过测试或安全扫描。
release 仍然要保留可追溯证据。
```

### 权衡 5：安全门禁和速度

```text
安全扫描会增加耗时，但生产链路需要可信制品。
所以 PR 阶段做快速检查，main/release 阶段做更完整的扫描和签名。
```

## 7. 回答“你遇到什么问题”

不要回答：

```text
没遇到什么问题。
```

准备真实问题：

- Docker build cache 没命中。
- `pull_request_target` 风险理解后避免使用。
- production values 用 tag 不够稳定，改为 digest。
- smoke test 只检查 health 不够，补了 version。
- workflow 权限最初过大，后来收紧。

回答结构：

```text
问题是什么。
怎么定位。
怎么修。
学到了什么。
```

## 8. 小练习

创建：

```text
portfolio/interview-notes.md
```

写入：

```markdown
# Interview Notes

## 2-minute Introduction

## 5-minute Architecture

## Key Trade-offs

## Problems and Fixes

## Metrics and Results

## Questions I Want to Ask Interviewers
```

## 9. 本节小结

你现在应该能把项目从“文件集合”讲成“工程故事”。

面试表达重点：

- 背景清楚。
- 任务明确。
- 行动具体。
- 结果可信。
- 权衡成熟。

