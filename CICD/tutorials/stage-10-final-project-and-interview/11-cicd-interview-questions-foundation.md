# 11：CI/CD 基础面试题

## 1. 本节目标

这一节整理基础题。

你不需要死背答案，但要能用自己的项目举例。

## 2. CI 和 CD 有什么区别？

参考答案：

```text
CI 是持续集成，重点是在代码变更进入主干前后自动运行测试、构建、静态检查和安全检查，尽早发现问题。
CD 可以指持续交付或持续部署。
持续交付强调制品随时可发布，但 production 可能需要人工审批。
持续部署强调通过自动化直接部署到生产。
```

项目例子：

```text
我的项目 PR 阶段是 CI，main 构建镜像并更新 staging 是 CD，production 通过 PR 审批属于持续交付。
```

## 3. 为什么要做分支保护？

参考答案：

```text
分支保护用于防止未检查、未 review 或未授权的变更进入主分支。
它通常要求 PR、required checks、review、禁止 force push。
CI/CD 依赖 main 分支作为可信输入，main 不受保护会影响后续制品和部署可信度。
```

## 4. 为什么本地命令要和 CI 一致？

参考答案：

```text
这样能减少本地和 CI 行为差异。
开发者可以在提交前用 make test、make lint、make vuln 复现 CI 主要检查。
CI 只是自动执行同一组命令。
```

## 5. Go 项目 CI 通常跑哪些检查？

参考答案：

```text
go fmt 或格式检查。
go test ./...
go test -race ./...
golangci-lint。
govulncheck。
构建二进制。
必要时运行集成测试和覆盖率报告。
```

## 6. `go test -race` 的作用是什么？

参考答案：

```text
它启用 Go race detector，用于发现并发读写数据竞争。
代价是运行更慢，所以可以在 main、nightly 或关键 PR 中运行，也可以根据项目规模决定是否每次 PR 都跑。
```

## 7. 为什么要用多阶段 Docker 构建？

参考答案：

```text
构建阶段需要 Go 编译器和依赖，运行阶段只需要编译后的二进制。
多阶段构建可以减小生产镜像体积，减少攻击面，也避免把源码和构建工具带进运行镜像。
```

## 8. 镜像 tag 和 digest 有什么区别？

参考答案：

```text
tag 是可读标签，可能被移动。
digest 是内容地址，指向具体镜像内容。
production 发布更推荐记录和使用 digest，便于审计、回滚和签名验证。
```

## 9. Kubernetes readiness 和 liveness 有什么区别？

参考答案：

```text
readiness 表示 Pod 是否准备好接收流量。
liveness 表示容器是否还活着，失败后 kubelet 会重启容器。
启动慢的服务还可以使用 startupProbe，避免启动阶段被 liveness 误杀。
```

## 10. Helm values 如何管理多环境？

参考答案：

```text
Chart 保存通用模板，values 保存环境差异。
例如 staging 和 production 使用不同 replicas、resources、ingress、image digest、环境变量。
CI 或 Argo CD 使用对应环境的 values 渲染和同步。
```

## 11. GitOps 是什么？

参考答案：

```text
GitOps 把 Git 作为期望状态来源。
CI 构建制品后更新部署仓库，Argo CD 等工具在集群内拉取 Git 并同步实际状态。
这样部署变更可以通过 PR、review、审计和 Git revert 管理。
```

## 12. Argo CD OutOfSync 和 Degraded 有什么区别？

参考答案：

```text
OutOfSync 表示集群实际状态和 Git 期望状态不一致。
Degraded 表示应用运行健康状态有问题。
一个应用可以 Synced 但 Degraded，也可以 OutOfSync 但暂时 Healthy。
```

## 13. Secret 在 CI 中怎么管理？

参考答案：

```text
secret 不应提交到 Git。
CI secret 应按环境隔离，production secret 应受 environment 审批保护。
尽量使用 OIDC 短期凭证替代长期云密钥。
workflow 权限要最小化，PR from fork 不应访问敏感 secret。
```

## 14. 什么是 SBOM 和 provenance？

参考答案：

```text
SBOM 描述制品包含哪些组件。
provenance 描述制品从哪个源码、哪个 workflow、哪个构建环境构建出来。
一个回答里面有什么，一个回答怎么来的。
```

## 15. 如何优化 CI 速度？

参考答案：

```text
先采集基线，例如 p50/p95、失败率和最慢 job。
常见优化包括 Go cache、Docker build cache、合理并行、路径过滤、取消过时 PR run、测试分层。
优化后要用数据对比，而不是凭感觉。
```

## 16. 小练习

把本节每个问题都用自己的项目回答一遍。

要求：

```text
每个答案至少包含一个 go-cicd-lab 中的具体例子。
```

## 17. 本节小结

基础题不是背概念。

最好答案是：

```text
概念解释 + 项目例子 + 权衡说明。
```

