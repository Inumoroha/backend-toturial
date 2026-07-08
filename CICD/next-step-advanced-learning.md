# CI/CD 进阶学习路线：下一步怎么学

> 你已经完成了从 CI/CD 全局心智、Git、Go 质量门禁、CI、Docker、CD、Kubernetes、Helm、GitOps、安全、可观测、优化、毕业项目到面试准备的完整路线。下一步不是继续机械堆工具，而是选择一个方向深入，把 CI/CD 能力放到真实生产系统里反复使用。

## 1. 先明确你的目标岗位

同样学 CI/CD，不同岗位的进阶重点不一样。

| 目标方向 | 你应该强化什么 |
| --- | --- |
| Go 后端工程师 | Go 工程化、测试、Docker、Kubernetes 基础、发布排障、可观测 |
| 中高级 Go 后端 | 服务可靠性、数据库迁移、性能优化、灰度发布、故障处理 |
| DevOps 工程师 | CI/CD 平台、云资源、Kubernetes、IaC、安全、监控 |
| 平台工程师 | 内部开发者平台、模板化、Golden Path、多团队交付效率 |
| SRE | SLO、告警、事故响应、容量、发布风险控制、自动化恢复 |
| 云原生工程师 | Kubernetes 深水区、Helm/Kustomize、GitOps、多集群、Service Mesh |
| DevSecOps | 供应链安全、SBOM、SLSA、签名、准入策略、合规审计 |

如果你当前目标是“Go 后端工程师”，推荐优先顺序：

1. Go 后端工程化深挖。
2. Kubernetes 生产实践。
3. 发布可靠性和可观测。
4. CI/CD 安全和供应链。
5. 平台工程和 IaC。

## 2. 进阶路线总图

```mermaid
flowchart LR
    A["Go 后端工程化"] --> B["Kubernetes 生产实践"]
    B --> C["发布工程与 GitOps"]
    C --> D["安全供应链"]
    D --> E["可观测与 SRE"]
    E --> F["平台工程与 IaC"]
    F --> G["真实项目沉淀"]
```

不要把它理解成严格线性。真实学习中可以交叉前进，但每次只选一个主线。

## 3. 进阶方向一：Go 后端工程化

### 为什么重要

CI/CD 的基础是项目本身足够工程化。

如果 Go 服务没有清晰结构、测试混乱、配置不规范、启动关闭不可靠，再强的流水线也只是把问题自动化放大。

### 学习重点

- 项目结构：`cmd/`、`internal/`、分层边界。
- 配置管理：环境变量、配置文件、默认值、校验。
- 错误处理：错误包装、错误分类、业务错误码。
- 日志：结构化日志、trace id、敏感信息过滤。
- HTTP 服务：超时、限流、CORS、中间件。
- graceful shutdown。
- 数据库连接池。
- migration 策略。
- 单元测试、集成测试、契约测试。
- 性能分析：pprof、benchmark。
- 并发安全：context、goroutine 生命周期、race detector。

### 推荐实战

把 `go-cicd-lab` 升级成更像真实服务：

- 增加 PostgreSQL migration。
- 增加 Redis 缓存。
- 增加分页、过滤和错误码。
- 增加结构化日志。
- 增加 graceful shutdown。
- 增加 pprof 只在非生产或受保护环境开放。
- 增加集成测试数据库容器。
- 增加 benchmark。

### 验收标准

- 能解释 Go 服务启动、健康检查、优雅关闭流程。
- 能说明测试分层策略。
- 能定位慢 SQL、慢接口和 goroutine 泄漏。
- 能把 Go 服务运行状态和 CI/CD 发布记录关联起来。

## 4. 进阶方向二：Kubernetes 生产实践

### 为什么重要

第 6 阶段学习的是 Kubernetes 入门和 Helm。

生产环境还要考虑：

- 高可用。
- 资源隔离。
- 安全边界。
- 弹性伸缩。
- 网络策略。
- 故障恢复。
- 成本控制。

### 学习重点

- requests/limits 设计。
- HPA、VPA、Cluster Autoscaler。
- PDB：PodDisruptionBudget。
- topology spread constraints。
- node affinity、pod anti-affinity。
- rollout 策略：RollingUpdate、maxSurge、maxUnavailable。
- readiness/liveness/startup probes 的生产配置。
- ConfigMap 和 Secret 更新策略。
- Service、Ingress、Gateway API。
- cert-manager 和 TLS。
- NetworkPolicy。
- RBAC 和 ServiceAccount。
- Pod Security Standards。
- Admission Controller。
- namespace 资源配额。

### 推荐实战

为 `go-cicd-lab` 增加：

- HPA。
- PDB。
- requests/limits。
- NetworkPolicy。
- Ingress TLS。
- ServiceMonitor。
- production namespace quota。
- 非 root + read-only root filesystem。

### 验收标准

- 能解释 Pod 为什么无法调度。
- 能解释服务为什么 Ready 但访问失败。
- 能解释 HPA 为什么没有扩容。
- 能解释 rolling update 中流量如何切换。
- 能排查 CrashLoopBackOff、ImagePullBackOff、OOMKilled。

## 5. 进阶方向三：发布工程与 Progressive Delivery

### 为什么重要

普通 CD 是“把新版本部署上去”。

发布工程关注的是：

```text
如何降低变更风险。
如何快速发现问题。
如何快速回滚。
如何让发布成为可控过程。
```

### 学习重点

- release train。
- staging -> production promotion。
- blue-green deployment。
- canary deployment。
- rolling deployment。
- feature flag。
- dark launch。
- database migration 与应用兼容。
- Argo Rollouts。
- Flagger。
- smoke test、synthetic check。
- 自动回滚条件。
- 发布冻结和变更窗口。

### 推荐实战

在 `go-cicd-lab` 上做：

- staging 自动发布。
- production PR 发布。
- canary 发布 10% -> 50% -> 100%。
- canary 期间观察错误率和延迟。
- 指标异常自动中止发布。
- feature flag 控制新接口。

### 验收标准

- 能解释 blue-green 和 canary 的区别。
- 能说明什么时候不适合自动部署到生产。
- 能设计一次有数据库变更的安全发布。
- 能说明如何回滚应用和数据库。

## 6. 进阶方向四：安全供应链与 DevSecOps

### 为什么重要

现代 CI/CD 不只是效率工具，也是供应链安全边界。

你要能回答：

```text
这个制品是谁构建的？
从哪个源码构建的？
里面有什么依赖？
是否被篡改？
能不能被部署？
```

### 学习重点

- threat modeling。
- secret scanning。
- dependency review。
- Dependabot/Renovate。
- govulncheck。
- CodeQL。
- Trivy/Grype。
- SBOM：SPDX、CycloneDX。
- SLSA。
- Sigstore/Cosign。
- keyless signing。
- artifact attestation。
- admission policy。
- OPA/Gatekeeper。
- Kyverno。
- least privilege。
- OIDC 短期凭证。
- audit log。

### 推荐实战

在 `go-cicd-lab` 增加：

- PR 依赖变更审查模板。
- 镜像必须签名才可部署。
- production 只允许指定仓库、指定 workflow 构建的镜像。
- 使用 Kyverno 或 Sigstore Policy Controller 做准入验证。
- 记录一次 secret 泄露演练。

### 验收标准

- 能解释 SBOM 和 provenance 的区别。
- 能解释为什么签名 digest，不签名 tag。
- 能解释 OIDC 如何替代长期云密钥。
- 能设计 secret 泄露响应流程。
- 能说明供应链安全门禁放在哪些阶段。

## 7. 进阶方向五：可观测与 SRE

### 为什么重要

发布后是否正常，不能只靠“workflow 绿了”。

SRE 关心：

- 用户是否受影响。
- 服务是否达成 SLO。
- 告警是否有效。
- 事故是否能快速恢复。
- 发布是否消耗错误预算。

### 学习重点

- Prometheus metrics。
- Grafana dashboard。
- Loki logs。
- OpenTelemetry traces。
- trace id 贯穿日志和请求。
- RED 指标：Rate、Errors、Duration。
- USE 指标：Utilization、Saturation、Errors。
- SLI/SLO/SLA。
- error budget。
- alert fatigue。
- incident management。
- postmortem。
- on-call。
- capacity planning。

### 推荐实战

为 `go-cicd-lab` 做：

- 请求量、错误率、p95 延迟 dashboard。
- 发布事件标注。
- 高错误率告警。
- 高延迟告警。
- Pod 重启告警。
- smoke test 失败告警。
- 一次故障演练和 postmortem。

### 验收标准

- 能写出关键 PromQL。
- 能解释 SLO 和错误预算。
- 能设计一个可行动告警。
- 能说明如何判断发布是否应该回滚。
- 能写事故时间线和 action items。

## 8. 进阶方向六：平台工程与内部开发者平台

### 为什么重要

当团队不止一个服务时，不能让每个项目都手写一套 CI/CD。

平台工程关注：

```text
如何为团队提供标准化、可复用、低认知负担的交付路径。
```

### 学习重点

- Golden Path。
- service template。
- reusable workflow。
- Helm chart template。
- Backstage。
- developer portal。
- self-service deploy。
- environment provisioning。
- policy as code。
- 多团队权限模型。
- 平台产品思维。

### 推荐实战

把 `go-cicd-lab` 抽象成模板：

- `template-go-service`。
- `template-github-actions`。
- `template-helm-chart`。
- `template-argocd-app`。
- 新服务 10 分钟内接入基础 CI/CD。

### 验收标准

- 能说明平台工程和传统 DevOps 的区别。
- 能设计新服务接入 CI/CD 的标准路径。
- 能解释模板化的收益和风险。
- 能说明如何避免平台变成“新瓶装旧审批”。

## 9. 进阶方向七：Infrastructure as Code

### 为什么重要

CI/CD 最后会碰到环境创建问题：

```text
集群怎么来？
数据库怎么来？
权限怎么来？
网络怎么来？
监控怎么来？
```

这些不应长期靠手点控制台。

### 学习重点

- Terraform。
- OpenTofu。
- Pulumi。
- Helm provider。
- Kubernetes provider。
- remote state。
- state lock。
- plan/apply 审批。
- drift detection。
- module 设计。
- 多环境目录结构。
- 云 IAM。

### 推荐实战

为 `go-cicd-lab` 写 IaC：

- 创建 namespace。
- 创建 registry 权限。
- 创建数据库。
- 创建 Secret Manager 条目。
- 创建 Kubernetes service account。
- 创建监控资源。

### 验收标准

- 能解释 Terraform state 是什么。
- 能解释 plan/apply 为什么要审批。
- 能说明 IaC drift 怎么处理。
- 能设计 staging/production 的 IaC 目录结构。

## 10. 进阶方向八：多服务与微服务交付

### 为什么重要

单服务 CI/CD 跑通后，真实公司常见问题是多服务。

你会遇到：

- monorepo。
- multi-repo。
- 共享库。
- 契约测试。
- 跨服务发布。
- 数据兼容。
- 版本矩阵。

### 学习重点

- monorepo path filter。
- affected services detection。
- service catalog。
- contract testing。
- consumer-driven contract。
- API versioning。
- backward compatibility。
- shared workflow。
- release orchestration。

### 推荐实战

把 `go-cicd-lab` 扩展成两个服务：

- `todo-api`。
- `notification-worker`。

要求：

- 独立镜像。
- 独立 Helm values。
- 共享 CI 模板。
- 修改一个服务只构建一个服务。
- 增加契约测试。

### 验收标准

- 能解释 monorepo 和 multi-repo 的 CI/CD 差异。
- 能设计只构建受影响服务的流水线。
- 能说明跨服务发布如何降低风险。

## 11. 进阶方向九：数据库变更与发布兼容

### 为什么重要

很多真实事故不是代码部署失败，而是数据库变更不可逆。

### 学习重点

- migration 工具。
- forward-compatible migration。
- expand/contract 模式。
- zero-downtime migration。
- 大表变更。
- 回滚策略。
- 数据备份。
- migration CI 校验。

### 推荐实战

做一次三步迁移：

1. 新增 nullable 字段。
2. 应用开始写新字段。
3. 回填数据并逐步移除旧字段。

### 验收标准

- 能解释为什么数据库 rollback 不总是安全。
- 能设计兼容旧版本和新版本的 schema 变更。
- 能说明 migration 在 CI/CD 中放在哪一步。

## 12. 进阶方向十：成本、效率与治理

### 为什么重要

成熟团队会关心：

- runner 成本。
- 镜像存储。
- artifact 保留。
- 集群资源。
- 闲置环境。
- 重复构建。

### 学习重点

- runner minutes。
- self-hosted runner。
- cache size。
- artifact retention。
- image lifecycle policy。
- Kubernetes resource quota。
- namespace cost allocation。
- FinOps。
- DORA 指标。
- delivery report。

### 推荐实战

为项目写：

- runner 成本估算。
- artifact retention 策略。
- 镜像清理策略。
- namespace 资源限制。
- 交付周报。

### 验收标准

- 能说明如何降低 CI 成本。
- 能解释为什么不能用成本优化牺牲质量门禁。
- 能用 DORA 指标描述交付效率。

## 13. 推荐 6 个月计划

### 第 1 个月：Go 服务生产化

- 完善测试、配置、日志、优雅关闭。
- 增加数据库 migration。
- 增加 `/metrics` 和 `/version`。

交付物：

- 一份 Go 服务工程化报告。

### 第 2 个月：Kubernetes 生产实践

- HPA、PDB、NetworkPolicy。
- 资源限制。
- Ingress TLS。
- rollout 策略。

交付物：

- 一份 Kubernetes production checklist。

### 第 3 个月：发布工程

- canary 或 blue-green。
- feature flag。
- 自动 smoke test。
- 回滚演练。

交付物：

- 一份 release playbook。

### 第 4 个月：供应链安全

- Cosign verification。
- admission policy。
- SBOM 管理。
- secret 泄露演练。

交付物：

- 一份 supply chain security report。

### 第 5 个月：可观测与 SRE

- dashboard。
- alert rules。
- SLO。
- incident drill。

交付物：

- 一份 SLO 和 incident report。

### 第 6 个月：平台化

- reusable workflows。
- service template。
- Helm chart template。
- 新服务接入文档。

交付物：

- 一个 Go 服务 CI/CD 模板仓库。

## 14. 推荐 10 个进阶项目

1. 做一个 Go 服务 CI/CD 模板仓库。
2. 做一个多服务 monorepo CI/CD。
3. 做一个 Argo Rollouts canary 示例。
4. 做一个 Kyverno 镜像签名准入策略。
5. 做一个 Terraform 创建测试环境的项目。
6. 做一个 GitHub Actions 指标采集 dashboard。
7. 做一个 SLO dashboard 和错误预算告警。
8. 做一个数据库 migration 零停机发布演练。
9. 做一个 Backstage 软件模板。
10. 做一个 incident drill 和 postmortem 示例库。

## 15. 学习方法建议

### 每次只解决一个真实问题

不要今天学 Terraform，明天学 Istio，后天学 Backstage。

更好的方式：

```text
我的 production 发布不可控，所以学 canary。
我的 secret 管理混乱，所以学 OIDC 和 Secret Manager。
我的 CI 太慢，所以学缓存和并发。
```

### 每个主题都要有产物

不要只写笔记。

产物可以是：

- workflow。
- Helm chart。
- dashboard。
- runbook。
- incident report。
- demo video。
- README。

### 每个月做一次复盘

复盘问题：

- 我解决了什么真实问题？
- 我新增了什么可展示产物？
- 哪个概念还讲不清？
- 下个月最值得学什么？

## 16. 你现在最推荐的下一步

如果你是以 Go 后端求职为目标，我建议下一步这样走：

1. 把 `go-cicd-lab` 真正做成一个 GitHub 作品集仓库。
2. 录一段 10 分钟演示视频。
3. 补 Kubernetes 生产实践：HPA、PDB、NetworkPolicy。
4. 做一次 canary 发布。
5. 做一次事故演练和 postmortem。
6. 把这些写进简历和面试故事。

这比继续开新课更有价值。

