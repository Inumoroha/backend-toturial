# Go 后端 CI/CD 面试题库

> 使用方式：不要死背。每道题最好都能用你的 `go-cicd-lab` 项目举例回答。好的回答通常是：先解释概念，再说明实践，再讲权衡，最后给排障或改进思路。

## 1. 项目介绍类

### 1. 请介绍一下你的 CI/CD 项目

答题要点：

- 项目业务是什么，例如 Go Todo API。
- 应用仓库负责什么：源码、测试、镜像、安全扫描。
- 部署仓库负责什么：Helm、values、Argo CD Application。
- PR 阶段做什么检查。
- main 阶段如何构建和推送镜像。
- staging 和 production 如何发布。
- 如何回滚。
- 如何观测和优化。

示例回答：

```text
我做了一个 Go 后端 CI/CD 毕业项目，业务是一个 Todo API，但重点是工程链路。
应用仓库负责 Go 代码、测试、Docker 镜像构建、漏洞扫描、SBOM 和签名。
部署仓库负责 Helm chart、staging/production values 和 Argo CD Application。
PR 阶段运行测试、lint 和 govulncheck，main 合并后构建镜像并推送 registry，staging 自动部署，production 通过部署仓库 PR 发布。
项目还包含 smoke test、/metrics、发布回滚 runbook 和流水线优化报告。
```

### 2. 这个项目最能体现你能力的地方是什么？

答题要点：

- 不要只列工具。
- 强调完整链路、权限边界、发布可靠性、排障和改进。
- 选 2 到 3 个亮点深入讲。

可以回答：

- 我把应用构建和部署状态分离成应用仓库与部署仓库。
- 我让 production 走 PR 和 digest，保证审计和可回滚。
- 我把安全、签名、SBOM 和可观测放进发布链路，而不是最后补文档。

### 3. 如果只给你 2 分钟，你会怎么讲这个项目？

答题结构：

1. 项目背景。
2. 技术链路。
3. 关键设计。
4. 结果和收获。

### 4. 你在项目中遇到过什么问题？

答题要点：

- 讲真实问题。
- 说明排查过程。
- 说明修复和预防。

可选例子：

- Docker build cache 不命中。
- workflow 权限最初过大。
- production values 最初只用 tag，后来改成 digest。
- smoke test 只检查 health，后来补 version 和关键 API。
- flaky test 影响 CI 信任。

### 5. 这个项目还有哪些不足？

答题要点：

- 诚实说明边界。
- 给出下一步改进。

示例：

```text
当前 production 是用命名空间模拟的，不是真实多区域生产环境。
告警规则有设计，但没有接入真实值班系统。
下一步我会补 Argo Rollouts canary、Kyverno 签名准入和 Terraform 环境创建。
```

## 2. CI/CD 基础概念

### 6. CI 是什么？

答题要点：

- Continuous Integration，持续集成。
- 目标是频繁合并代码并自动验证。
- 常见验证：测试、lint、构建、安全扫描。
- 重点是尽早发现问题。

### 7. CD 是什么？

答题要点：

- Continuous Delivery：持续交付，制品随时可发布，生产可能需要人工审批。
- Continuous Deployment：持续部署，通过自动化直接部署到生产。
- 两者区别在 production 是否自动发布。

### 8. CI 和 CD 的边界是什么？

答题要点：

- CI 关注代码变更是否可集成。
- CD 关注可信制品如何进入环境。
- CI 输出制品，CD 消费制品。
- 权限边界不同：PR CI 不应拿 production secret。

### 9. 为什么需要 CI/CD？

答题要点：

- 减少手工操作。
- 提高反馈速度。
- 提高发布一致性。
- 降低部署风险。
- 保留审计记录。
- 支持快速回滚。

### 10. CI/CD 常见阶段有哪些？

答题要点：

- checkout。
- setup runtime。
- dependency install。
- test。
- lint。
- security scan。
- build artifact。
- build image。
- push registry。
- deploy staging。
- smoke test。
- promote production。
- observe。
- rollback。

### 11. 什么是 pipeline、workflow、job、step、runner？

答题要点：

- pipeline/workflow 是一次自动化流程。
- job 是一组步骤，通常在一个 runner 上运行。
- step 是具体命令或 action。
- runner 是执行环境。
- 不同平台术语略有差异。

### 12. 什么是 artifact？

答题要点：

- 流水线产生的可保存产物。
- 例子：测试报告、覆盖率、二进制、SBOM、扫描报告。
- artifact 不应包含 secret。

### 13. 什么是 environment？

答题要点：

- 代表部署环境，例如 staging、production。
- 可绑定 secret、审批、分支限制。
- production environment 应有更严格保护。

### 14. 什么是 promotion？

答题要点：

- 将同一个可信制品从一个环境推进到下一个环境。
- 推荐推进 digest，而不是重新构建一次。
- 避免 staging 测的是 A，production 发的是 B。

### 15. 什么是 immutable artifact？

答题要点：

- 构建后内容不可变的制品。
- 容器镜像 digest 是典型例子。
- 有利于审计、回滚和签名验证。

## 3. Git、分支与协作

### 16. Git merge 和 rebase 有什么区别？

答题要点：

- merge 保留分支合并历史。
- rebase 重新整理提交历史，让线性更清晰。
- 公共分支谨慎 rebase。
- 团队要有统一策略。

### 17. 什么是 trunk-based development？

答题要点：

- 开发者频繁合并到主干。
- 分支生命周期短。
- 依赖自动化测试和 feature flag。
- 适合高频交付。

### 18. Git Flow 和 trunk-based 如何选择？

答题要点：

- Git Flow 分支较多，适合发布周期明确或版本维护复杂场景。
- trunk-based 更强调小步提交和快速集成。
- CI/CD 成熟后更倾向 trunk-based。

### 19. 为什么要做 branch protection？

答题要点：

- 防止未经检查的代码进入 main。
- 要求 PR、review、required checks。
- 禁止 force push。
- main 是后续构建和发布的可信输入。

### 20. 什么是 CODEOWNERS？

答题要点：

- 定义某些路径的默认 reviewer。
- 常用于 workflow、production values、Helm chart、RBAC。
- 可以保护关键目录变更。

### 21. tag 和 release 在 CI/CD 中有什么作用？

答题要点：

- tag 标识版本点。
- release 可关联 changelog、制品和说明。
- release workflow 可由 tag 触发。
- 生产发布不要只依赖可移动 tag，应记录 digest。

### 22. 如何设计 PR 模板？

答题要点：

- 变更说明。
- 测试说明。
- 风险说明。
- 回滚方式。
- 是否修改依赖、workflow、secret、部署配置。

### 23. 如何处理 hotfix？

答题要点：

- 从稳定版本或 main 创建 hotfix 分支。
- 保持 CI 检查。
- 优先小变更。
- 发布后补回主干。
- 记录事故和根因。

## 4. Go 项目质量门禁

### 24. Go 项目 CI 应该跑哪些命令？

答题要点：

- `go test ./...`。
- `go test -race ./...`。
- `go vet ./...`。
- `golangci-lint run`。
- `govulncheck ./...`。
- `go build ./cmd/api`。

### 25. `go test ./...` 有什么作用？

答题要点：

- 运行当前模块下所有 package 的测试。
- 是 Go 项目最基础的质量门禁。
- CI 中必须稳定可复现。

### 26. `go test -race` 解决什么问题？

答题要点：

- 检测数据竞争。
- 对并发服务很重要。
- 运行更慢，可以放在 main、nightly 或关键 PR。

### 27. `go vet` 和 golangci-lint 有什么区别？

答题要点：

- `go vet` 是 Go 官方静态检查工具。
- golangci-lint 聚合多个 linter。
- golangci-lint 更可配置，适合团队规范。

### 28. `govulncheck` 和普通依赖漏洞扫描有什么不同？

答题要点：

- `govulncheck` 会结合 Go 漏洞数据库和调用路径。
- 不只看依赖存在，还看是否实际调用有漏洞符号。
- 适合 Go 项目安全门禁。

### 29. Go 项目如何做集成测试？

答题要点：

- 使用 Docker Compose 或 testcontainers 准备数据库。
- 用 build tags 区分 integration。
- PR 跑轻量测试，main/nightly 跑完整集成测试。
- 数据要隔离，测试后清理。

### 30. 如何处理 flaky test？

答题要点：

- 记录失败 test、run id、commit、重跑结果。
- 分类原因：时间依赖、并发、共享状态、外部服务。
- 短期可隔离或重试，长期必须修复。
- 不要让 rerun 成为默认文化。

### 31. 如何提高 Go 测试速度？

答题要点：

- 使用 Go test cache。
- 区分 unit/integration/e2e。
- 避免不必要外部依赖。
- 并行 package 测试。
- 优化慢测试。
- 记录耗时最长的 test。

### 32. 如何在 Go 二进制里注入版本信息？

答题要点：

- 使用 `-ldflags "-X main.version=... -X main.commit=..."`。
- 通过 `/version` 暴露。
- 发布后可反查运行版本。

### 33. Go 服务为什么需要 graceful shutdown？

答题要点：

- Kubernetes rolling update 时旧 Pod 会收到终止信号。
- 服务应停止接收新请求，并给已有请求时间完成。
- 避免发布时用户请求被中断。

### 34. Go 服务健康检查怎么设计？

答题要点：

- `/healthz`：进程活着。
- `/readyz`：是否准备好接收流量，通常检查依赖。
- `/metrics`：Prometheus 指标。
- `/version`：版本和 commit。

## 5. GitHub Actions / GitLab CI

### 35. GitHub Actions 的 workflow、job、step 分别是什么？

答题要点：

- workflow 是完整流水线。
- job 是可并行或依赖执行的任务。
- step 是 job 内的命令或 action。
- runner 执行 job。

### 36. `GITHUB_TOKEN` 是什么？

答题要点：

- GitHub 自动提供给 workflow 的 token。
- 权限受 repository、workflow `permissions` 和 event 影响。
- 应显式设置最小权限。

### 37. 为什么要设置 `permissions: contents: read`？

答题要点：

- 最小权限基线。
- 大多数 PR CI 只需要读取代码。
- 写 packages、attestations、id-token 等权限只给必要 job。

### 38. `pull_request` 和 `pull_request_target` 有什么区别？

答题要点：

- `pull_request` 在 PR 上下文运行，权限相对受限。
- `pull_request_target` 在目标仓库上下文运行，可能访问更多权限和 secrets。
- 不应在 `pull_request_target` 中 checkout 并执行不可信 PR 代码。

### 39. 如何防止 workflow 注入？

答题要点：

- 不把 PR title/body/comment 直接拼进 shell。
- 使用环境变量传递不可信输入。
- shell 中加引号。
- 避免执行来自 artifact 或 PR 的脚本。

### 40. 第三方 action 有什么风险？

答题要点：

- 它是供应链依赖。
- 分支或 tag 可能变化。
- action 可能访问 workspace 和 secrets。
- 关键 workflow 应 pin 到 commit SHA，并通过 Dependabot 更新。

### 41. 如何使用 cache？

答题要点：

- 先使用官方 setup action 内置缓存。
- 手动 cache 要设计 key 和 restore key。
- 不要缓存 secret。
- 观察 cache hit/miss。

### 42. `needs` 有什么作用？

答题要点：

- 表达 job 依赖关系。
- 例如 image job 依赖 test/lint/vuln。
- 避免不可信或未验证代码进入后续构建。

### 43. matrix 用在什么场景？

答题要点：

- 多 Go 版本。
- 多 OS。
- 多数据库版本。
- 多服务模块。
- 但要控制组合爆炸。

### 44. concurrency 有什么作用？

答题要点：

- 控制同组 workflow 并发。
- PR 可以 `cancel-in-progress: true` 取消旧 run。
- production deploy 应串行，通常不取消进行中的发布。

### 45. 如何让 CI 失败更容易排查？

答题要点：

- 拆分 job 和 step。
- 使用日志分组。
- 输出 job summary。
- 失败时上传测试报告。
- 使用 `gh run view --log-failed`。

### 46. GitLab CI 和 GitHub Actions 有什么差异？

答题要点：

- GitLab 使用 stages/jobs，GitHub 使用 workflow/jobs/steps。
- GitLab CI/CD 与 GitLab repo、registry、environment 集成强。
- GitHub Actions Marketplace 丰富。
- 核心思想相同：触发、runner、job、artifact、cache、environment。

## 6. Docker、镜像与 Registry

### 47. Dockerfile 多阶段构建的好处是什么？

答题要点：

- 构建阶段包含编译工具。
- 运行阶段只保留二进制和必要文件。
- 减小镜像体积和攻击面。
- 提高生产镜像可控性。

### 48. 为什么 Dockerfile 要先复制 `go.mod` 和 `go.sum`？

答题要点：

- Docker 按层缓存。
- 依赖文件不变时可复用 `go mod download` 层。
- 如果先 `COPY . .`，任何源码变化都会让依赖层失效。

### 49. `.dockerignore` 有什么作用？

答题要点：

- 减小 build context。
- 避免无关文件进入构建。
- 降低 secret 误入镜像风险。
- 提高构建速度。

### 50. 为什么生产容器不建议 root 运行？

答题要点：

- 降低容器逃逸或应用漏洞后的影响。
- Kubernetes securityContext 可进一步限制权限。
- 结合 drop capabilities、readOnlyRootFilesystem 更好。

### 51. scratch、distroless、alpine 怎么选？

答题要点：

- scratch 最小，但排障和证书处理麻烦。
- distroless 小且适合生产，但没有 shell。
- alpine 小但 musl 兼容性要注意。
- slim 兼容好但更大。
- Go 静态二进制常用 distroless 或 scratch。

### 52. 镜像 tag 策略怎么设计？

答题要点：

- `sha-<commit>` 用于可追溯。
- semver 用于 release。
- `main` 或 `latest` 可用于开发便利，但不应用作 production 唯一依据。
- production 记录 digest。

### 53. digest 为什么重要？

答题要点：

- digest 指向不可变内容。
- tag 可能移动。
- 签名、SBOM、provenance 和部署审计都应围绕 digest。

### 54. 如何扫描镜像漏洞？

答题要点：

- 使用 Trivy、Grype 或平台扫描。
- 扫描系统包、应用依赖、配置。
- HIGH/CRITICAL 可阻断。
- 无修复版本时记录风险接受。

### 55. 镜像扫描今天失败昨天通过，说明什么？

答题要点：

- 漏洞数据库更新后旧镜像可能发现新漏洞。
- 不一定是 CI 坏了。
- 应查看 CVE、严重度、修复版本和实际影响。

### 56. 如何优化 Docker build 速度？

答题要点：

- 调整 Dockerfile 层顺序。
- `.dockerignore`。
- BuildKit cache mount。
- buildx `cache-from/cache-to`。
- PR 阶段减少多平台构建。

## 7. CD、环境与发布

### 57. staging 和 production 有什么区别？

答题要点：

- staging 用于发布前验证，自动化程度更高。
- production 面向用户，审批、审计、回滚要求更高。
- secret、数据、权限、资源隔离。

### 58. 为什么 production 不建议直接由 PR workflow 部署？

答题要点：

- PR 代码不可信。
- production secret 风险高。
- 应由受保护分支、受保护环境或部署仓库 PR 触发。

### 59. Continuous Delivery 和 Continuous Deployment 怎么落地？

答题要点：

- Delivery：自动构建、测试、准备发布，production 人工审批。
- Deployment：通过自动化直接到生产。
- 项目中 staging 可自动部署，production 采用持续交付。

### 60. 什么是 smoke test？

答题要点：

- 发布后最小可用验证。
- 检查 health、ready、version、关键 API。
- 不是完整 e2e。
- 失败应阻断或回滚。

### 61. 如何设计回滚？

答题要点：

- 记录当前和上一个可信 digest。
- GitOps 中优先 Git revert 部署仓库。
- 保留 rollback runbook。
- 回滚后执行 smoke test 和观测。

### 62. 什么情况下应该回滚？

答题要点：

- smoke test 失败。
- 5xx 错误率超过阈值。
- p95 延迟异常。
- Pod 大量重启。
- 关键业务指标下降。
- 数据库迁移失败。

### 63. 如何处理数据库迁移和应用发布？

答题要点：

- 使用向前兼容迁移。
- expand/contract 模式。
- 先加字段，再部署代码，再清理旧字段。
- 大表变更需评估锁表和回滚。
- migration 要在 staging 先验证。

### 64. 如何做生产审批？

答题要点：

- GitHub environments required reviewers。
- 部署仓库 PR。
- CODEOWNERS。
- change window。
- release checklist。
- 审批不是形式，必须看 diff、digest、回滚方案。

## 8. Kubernetes 与 Helm

### 65. Pod、Deployment、Service 分别是什么？

答题要点：

- Pod 是最小运行单元。
- Deployment 管理副本和滚动更新。
- Service 提供稳定访问入口和负载均衡。

### 66. ConfigMap 和 Secret 有什么区别？

答题要点：

- ConfigMap 保存非敏感配置。
- Secret 保存敏感配置，但 Kubernetes Secret 默认只是 base64，不等于强加密。
- 真实生产可结合 External Secrets、Vault、云 Secret Manager。

### 67. readinessProbe 和 livenessProbe 区别？

答题要点：

- readiness 决定是否接收流量。
- liveness 决定是否重启容器。
- startupProbe 用于启动慢的服务。

### 68. Pod CrashLoopBackOff 怎么排查？

答题要点：

- `kubectl logs`。
- `kubectl describe pod`。
- 看 exit code、事件、环境变量、配置、依赖。
- 检查 liveness 是否误杀。
- 检查镜像、命令和资源限制。

### 69. ImagePullBackOff 怎么排查？

答题要点：

- 镜像名和 tag/digest 是否正确。
- registry 是否可访问。
- imagePullSecrets。
- 权限。
- 镜像架构。

### 70. OOMKilled 怎么排查？

答题要点：

- 查看容器 last state。
- 检查 memory limit。
- 查看应用内存曲线。
- Go 服务可用 pprof。
- 调整 requests/limits 或修复泄漏。

### 71. Helm 解决什么问题？

答题要点：

- 模板化 Kubernetes manifests。
- values 管理环境差异。
- release 管理和渲染。
- 适合复用和多环境部署。

### 72. Helm values 如何设计？

答题要点：

- 默认 values 保存通用配置。
- staging/production 覆盖环境差异。
- Secret 不明文放 values。
- production 使用 digest。

### 73. Helm template 和 helm install 有什么区别？

答题要点：

- `helm template` 本地渲染 YAML，不连接集群。
- `helm install/upgrade` 会操作集群。
- CI 中常用 template/lint 做校验。

### 74. Kubernetes 中如何做资源限制？

答题要点：

- requests 用于调度保证。
- limits 是最大限制。
- CPU 可被 throttling，内存超限会 OOMKilled。
- 需要结合监控调优。

### 75. HPA 扩容不生效怎么排查？

答题要点：

- metrics-server 是否正常。
- HPA target 指标是否存在。
- Pod requests 是否设置。
- 当前指标是否超过阈值。
- maxReplicas 是否限制。

## 9. GitOps 与 Argo CD

### 76. GitOps 是什么？

答题要点：

- Git 是期望状态来源。
- 控制器持续对比 Git 和集群。
- 通过 PR 管理部署变更。
- 有利于审计和回滚。

### 77. push-based CD 和 pull-based GitOps 区别？

答题要点：

- push-based：CI 直接连接集群部署。
- pull-based：集群内控制器从 Git 拉取期望状态。
- GitOps 减少 CI 持有集群高权限的需求。

### 78. Argo CD Application 是什么？

答题要点：

- 定义从哪个 repo/path/revision 同步到哪个 cluster/namespace。
- 包含 sync policy。
- 是 Argo CD 管理应用的核心对象。

### 79. AppProject 有什么作用？

答题要点：

- 限制 Application 可用 sourceRepos。
- 限制 destination cluster/namespace。
- 限制资源类型。
- 是多团队隔离和安全边界。

### 80. OutOfSync 怎么处理？

答题要点：

- 看 diff。
- 判断是 Git 更新未同步，还是集群 drift。
- 正常变更可 sync。
- 手动 drift 需要决定保留还是恢复，并补回 Git。

### 81. Synced 但服务不可用说明什么？

答题要点：

- Synced 只说明资源和 Git 一致。
- 不代表业务可用。
- 继续看 Health、Pod 状态、Service endpoints、Ingress、日志和指标。

### 82. prune 有什么风险？

答题要点：

- Git 中删除资源后，prune 会删除集群资源。
- 误删模板或 values 可能导致资源被删。
- production 开启 prune 要谨慎，有 review 和恢复方案。

### 83. selfHeal 有什么风险？

答题要点：

- 会自动纠正手动修改。
- 防止 drift，但可能覆盖紧急手动修复。
- 紧急修复必须补回 Git。

### 84. GitOps 如何回滚？

答题要点：

- 首选 Git revert 部署仓库 commit。
- 或改回旧 image digest。
- 然后 Argo CD sync。
- 避免只在集群手动回滚。

## 10. 安全与供应链

### 85. CI/CD 的主要攻击面有哪些？

答题要点：

- 源码仓库。
- workflow。
- runner。
- secrets。
- dependencies。
- artifacts。
- container registry。
- deploy repo。
- Kubernetes cluster。
- third-party actions。

### 86. secret 泄露后怎么办？

答题要点：

- 立即撤销或轮换。
- 暂停相关 workflow。
- 检查 Git 历史、logs、artifact、镜像层。
- 查看访问日志。
- 重新发布干净制品。
- 清理历史和缓存。
- 增加 secret scanning 和 runbook。

### 87. 为什么删除 Git 历史不够？

答题要点：

- secret 可能已被复制到远端、日志、缓存、他人本地。
- 删除历史不能让 secret 失效。
- 轮换才是关键。

### 88. OIDC 为什么比长期密钥好？

答题要点：

- 不需要把长期云密钥放到 CI secret。
- workflow 使用短期身份换取短期凭证。
- 云侧可限制 repository、branch、environment。
- 泄露影响窗口更小。

### 89. SBOM 是什么？

答题要点：

- Software Bill of Materials。
- 描述制品包含的组件、依赖、版本。
- 用于漏洞影响分析、审计和透明度。

### 90. provenance 是什么？

答题要点：

- 构建来源证明。
- 描述哪个仓库、commit、workflow、runner 构建了哪个 digest。
- 回答制品从哪里来、怎么构建。

### 91. SLSA 是什么？

答题要点：

- 供应链安全框架。
- 关注源码保护、构建可追溯、provenance、构建环境可信。
- 不是单个工具。

### 92. Cosign keyless signing 是什么？

答题要点：

- 使用 OIDC 短期身份签名制品。
- 不需要长期私钥。
- 验证时检查 digest、签名、identity、issuer。

### 93. 为什么只验证“有签名”不够？

答题要点：

- 任何人都可能签名自己的镜像。
- 必须验证签名身份是否来自可信仓库、workflow、branch/environment。
- 还要验证 issuer。

### 94. 如何做 admission policy？

答题要点：

- 使用 Kyverno、OPA/Gatekeeper、Sigstore Policy Controller。
- 限制 registry。
- 要求 digest。
- 要求签名。
- 验证签名 identity。
- 禁止 privileged、root、latest tag。

### 95. 什么是最小权限？

答题要点：

- 只给完成任务所需的最少权限。
- workflow、job、token、cloud role、kubeconfig 都适用。
- 权限扩大必须有理由和审批。

### 96. 如何保护第三方 action？

答题要点：

- 选择可信维护者。
- pin 到版本或 commit SHA。
- Dependabot 更新。
- 避免给不必要 secrets。
- 限制组织允许的 actions。

## 11. 可观测、优化与 DORA

### 97. CI/CD 可观测性看哪些信号？

答题要点：

- workflow logs。
- job duration。
- success/failure/cancelled。
- cache hit。
- test report。
- image digest。
- deployment event。
- smoke test result。
- application metrics。

### 98. 如何分析 CI 变慢？

答题要点：

- 先建立 p50/p95 基线。
- 找最慢 workflow 和 job。
- 拆 setup、dependency、test、build、scan、deploy。
- 看最近变更。
- 看 cache 是否命中。
- 提出假设并验证。

### 99. 如何优化 Go CI？

答题要点：

- setup-go cache。
- 固定工具版本。
- 测试分层。
- 并行 job。
- 避免每次安装大型工具。
- 取消过时 PR run。

### 100. 如何优化 Docker build？

答题要点：

- Dockerfile 层顺序。
- `.dockerignore`。
- BuildKit cache mount。
- buildx GitHub Actions cache。
- 减少 PR 多平台构建。

### 101. p50、p95 分别表示什么？

答题要点：

- p50 表示中位数，多数体验。
- p95 表示长尾体验。
- 平均值可能被极端值影响。
- 优化 CI 要看 p50 和 p95。

### 102. DORA 四项指标是什么？

答题要点：

- Deployment Frequency。
- Lead Time for Changes。
- Change Failure Rate。
- Time to Restore Service。

### 103. DORA 指标怎么定义？

答题要点：

- 先定义什么算 production deployment。
- 先定义什么算 failed change。
- lead time 可从 merge main 到 production 部署完成。
- MTTR 从故障发现到恢复。

### 104. SLO 是什么？

答题要点：

- Service Level Objective。
- 例如 30 天内 99.9% 请求成功。
- 错误预算用于指导发布节奏。
- 发布消耗错误预算时要收敛变更。

### 105. 什么是可行动告警？

答题要点：

- 告警要说明影响、当前值、阈值、可能原因、runbook。
- 不要为无影响抖动告警。
- 使用 `for` 降低噪音。
- PR 失败不应叫醒值班。

### 106. Go 服务应该暴露哪些指标？

答题要点：

- 请求量。
- 错误率。
- 请求耗时。
- in-flight 请求。
- 数据库错误。
- 业务成功/失败数。
- runtime 指标。

### 107. label cardinality 为什么重要？

答题要点：

- 高 cardinality 会产生大量时间序列。
- 不要把 user id、request id、原始 URL、错误全文作为 label。
- route 用模板，例如 `/users/{id}`。

## 12. 排障场景题

### 108. PR CI 失败，你第一步看什么？

答题要点：

- 哪个 workflow。
- 哪个 job。
- 哪个 step。
- 最后关键日志。
- 是否可复现。
- 最近改过什么。

### 109. main 失败但 PR 通过，怎么排查？

答题要点：

- 比较 main 和 PR workflow 差异。
- main 可能多了 image build、deploy、安全扫描。
- 检查权限和 secrets。
- 检查 event context。

### 110. deployment 成功但服务访问失败，怎么查？

答题要点：

- Pod 状态。
- Service endpoints。
- Ingress。
- readiness。
- logs。
- config/secret。
- network policy。
- 应用监听地址和端口。

### 111. Argo CD 显示 Healthy，但用户仍报错，怎么办？

答题要点：

- Healthy 是资源健康，不代表所有业务路径。
- 查 smoke test、业务日志、metrics、trace。
- 看版本和最近发布。
- 需要时回滚。

### 112. workflow 无法推送 GHCR 镜像怎么办？

答题要点：

- 检查 `packages: write`。
- 检查 docker login。
- 检查 registry 地址。
- 检查 package visibility。
- 检查 repository settings。

### 113. OIDC AssumeRole 失败怎么办？

答题要点：

- workflow 是否有 `id-token: write`。
- trust policy 是否匹配 repo/ref/environment。
- audience 是否正确。
- branch/tag 是否符合。
- 云角色权限是否足够。

### 114. Dependabot PR 很多，怎么处理？

答题要点：

- 分组 minor/patch。
- 限制 open PR 数量。
- 自动跑测试。
- 高危漏洞优先。
- 不建议无脑自动合并。

### 115. 镜像签名验证失败怎么办？

答题要点：

- 确认验证的是 digest。
- 检查签名是否推送到 registry。
- 检查 certificate identity。
- 检查 OIDC issuer。
- 检查 workflow/ref 是否符合策略。

### 116. Helm template 正常，部署后不正常，怎么查？

答题要点：

- template 只验证渲染。
- 继续查 Kubernetes events、Pod logs、runtime config。
- 检查 values 是否环境正确。
- 检查 Secret 和依赖服务。

### 117. Kubernetes rollout 卡住怎么办？

答题要点：

- `kubectl rollout status`。
- `kubectl describe deploy/pod`。
- readiness 未通过。
- 镜像拉取失败。
- 资源不足。
- 新 Pod 起不来。
- maxUnavailable/maxSurge 配置。

### 118. 生产事故中先排查还是先回滚？

答题要点：

- 先判断影响范围和回滚信号。
- 如果用户影响严重且与最近发布高度相关，优先回滚止血。
- 恢复后再深入定位。
- 记录时间线。

### 119. CI 偶发网络失败怎么办？

答题要点：

- 判断是外部依赖、runner、registry、Go proxy。
- 增加合理重试。
- 使用缓存或内部代理。
- 区分基础设施失败和代码失败。
- 不要用无限重试掩盖问题。

### 120. runner 排队很久怎么办？

答题要点：

- 查看 queue time。
- 减少无效 run。
- concurrency cancel old PR。
- path filters。
- self-hosted runner。
- 调整并行度。
- 成本评估。

## 13. 架构设计题

### 121. 从零设计 Go 项目的 CI/CD，你怎么做？

答题要点：

1. 本地命令标准化。
2. PR CI：test/lint/vuln。
3. main 构建镜像。
4. 推送 registry。
5. staging 自动部署。
6. production 审批发布。
7. 安全扫描、SBOM、签名。
8. 可观测和回滚。

### 122. 如何设计多环境发布？

答题要点：

- dev/staging/production。
- 配置分离。
- secret 分离。
- 权限分离。
- staging 自动，production 审批。
- 同一 digest promotion。

### 123. 如何设计 monorepo CI/CD？

答题要点：

- path filter 检测受影响服务。
- 每个服务独立测试和构建。
- 共享 reusable workflow。
- 共享基础镜像和 chart 模板。
- 避免全量构建。

### 124. 如何设计多团队 GitOps？

答题要点：

- AppProject 隔离 repo/namespace。
- RBAC。
- CODEOWNERS。
- namespace quota。
- policy as code。
- 平台团队维护模板，业务团队维护 values。

### 125. 如何设计 production secret 管理？

答题要点：

- 不明文进 Git。
- 使用 Secret Manager/Vault/External Secrets/SOPS。
- 环境隔离。
- 最小权限。
- 轮换。
- 审计。

### 126. 如何设计可回滚发布？

答题要点：

- 每次发布记录 digest。
- 部署仓库 PR。
- Git revert。
- 旧版本保留。
- smoke test。
- 回滚信号和 runbook。

### 127. 如何设计流水线权限模型？

答题要点：

- PR 只读。
- main image job 可写 packages。
- signing job 有 id-token。
- deploy job 只写 deploy repo。
- production environment 需要审批。
- 云权限使用 OIDC。

### 128. 如何设计发布证据链？

答题要点：

- commit SHA。
- workflow run URL。
- image digest。
- SBOM。
- provenance。
- signature verification。
- deploy repo commit。
- Argo CD history。
- smoke test result。

### 129. 如何设计高可靠发布系统？

答题要点：

- 小批量变更。
- 自动测试。
- canary。
- 指标监控。
- 自动或快速回滚。
- release checklist。
- 事故复盘。
- 错误预算。

### 130. 如何设计开发者友好的平台能力？

答题要点：

- 标准模板。
- 自助创建服务。
- reusable workflow。
- 默认安全基线。
- 默认观测。
- 文档清晰。
- 减少业务团队认知负担。

## 14. 行为面试和沟通题

### 131. 你如何推动团队接入 CI/CD？

答题要点：

- 先找痛点。
- 从最小可用流水线开始。
- 不一次引入太多门禁。
- 用数据证明收益。
- 提供模板和文档。
- 收集团队反馈。

### 132. 开发抱怨 CI 太慢怎么办？

答题要点：

- 先承认反馈。
- 用数据分析瓶颈。
- 区分必要检查和可优化部分。
- 优化缓存、并行、路径过滤。
- 不牺牲关键安全和质量门禁。

### 133. 团队想跳过测试紧急上线怎么办？

答题要点：

- 评估风险和影响。
- 是否有紧急流程和审批。
- 保留最低限度 smoke test。
- 记录例外原因。
- 事后补测试和复盘。

### 134. 你如何处理和安全团队的分歧？

答题要点：

- 理解安全要求背后的风险。
- 提供替代方案。
- 用风险等级和影响范围讨论。
- 记录风险接受。
- 共同制定自动化门禁。

### 135. 你如何写一份事故复盘？

答题要点：

- summary。
- timeline。
- impact。
- root cause。
- detection。
- response。
- what went well。
- what went wrong。
- action items。
- 避免责备个人，关注系统改进。

### 136. 如果你不会某个工具，面试中怎么回答？

答题要点：

- 诚实说明没有生产经验。
- 说明理解相邻概念。
- 说明会如何学习和验证。
- 用已做过的类似工具类比。

示例：

```text
我没有在生产中用过 Argo Rollouts，但我理解 canary 的核心是分批流量、指标判断和自动回滚。我在项目中做过 Argo CD GitOps 和 smoke test，下一步会用 Argo Rollouts 把这部分补成渐进式发布。
```

## 15. 手写或实操题

### 137. 写一个最小 GitHub Actions Go CI

要点：

- checkout。
- setup-go。
- cache。
- go test。
- permissions read。

### 138. 写一个 Dockerfile

要点：

- 多阶段构建。
- 先 copy go.mod/go.sum。
- 非 root 运行。
- 注入 commit。

### 139. 写一个 Helm Deployment 模板

要点：

- image repository/tag/digest。
- resources。
- probes。
- env。
- securityContext。

### 140. 写一个 smoke test 脚本

要点：

- `set -euo pipefail`。
- BASE_URL 必填。
- curl health/ready/version。
- 失败返回非 0。

### 141. 写一个 Dependabot 配置

要点：

- gomod。
- github-actions。
- weekly。
- open PR limit。
- grouping。

### 142. 写一个 production PR checklist

要点：

- image digest。
- scan passed。
- signature verified。
- SBOM/provenance。
- smoke test plan。
- rollback plan。
- owner approval。

## 16. 面试最后反问

你可以问：

- 团队现在的发布频率是多少？
- production 发布是否自动化？
- 回滚通常怎么做？
- CI/CD 最大痛点是速度、稳定性、安全还是协作？
- 是否使用 Kubernetes、Helm、GitOps？
- secret 和环境权限怎么管理？
- 是否有 SLO 和事故复盘机制？
- 平台团队和业务团队如何分工？
- 新服务接入 CI/CD 需要多久？
- 团队希望这个岗位解决什么交付问题？

好的反问能让面试官感受到你关心真实工程，而不是只背工具。

## 17. 总复习方法

按下面顺序准备：

1. 背熟 2 分钟项目介绍。
2. 画出 PR 到 production 的链路。
3. 准备 5 个权衡。
4. 准备 5 个故障排查故事。
5. 准备 3 个安全场景。
6. 准备 3 个优化数据或优化计划。
7. 做一次模拟面试。

最终目标不是答出所有题，而是让面试官相信：

```text
你能把一个 Go 后端项目从代码提交，稳定、安全、可观测地交付到环境中。
```

