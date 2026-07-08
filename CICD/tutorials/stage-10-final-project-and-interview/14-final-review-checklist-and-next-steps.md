# 14：最终复习清单与后续路线

## 1. 本节目标

这是整条 CI/CD 学习路线的收尾。

你要确认：

```text
项目能运行。
流水线能解释。
发布能演示。
回滚能执行。
问题能排查。
面试能讲清。
```

## 2. 最终项目清单

### 应用仓库

- [ ] Go API 可本地运行。
- [ ] 有 `/healthz`、`/readyz`、`/version`、`/metrics`。
- [ ] 有单元测试。
- [ ] 有集成测试或集成测试设计。
- [ ] 有 Makefile。
- [ ] 有 Dockerfile。
- [ ] 有 `.dockerignore`。
- [ ] 有 `ci.yml`。
- [ ] 有 `image.yml`。
- [ ] 有安全扫描。
- [ ] 有 SBOM/provenance/签名或设计说明。
- [ ] 有 smoke test。
- [ ] 有 README 和 docs。

### 部署仓库

- [ ] 有 Helm chart。
- [ ] 有 staging values。
- [ ] 有 production values。
- [ ] 有 Argo CD Application。
- [ ] 有 AppProject。
- [ ] production 通过 PR 发布。
- [ ] 有回滚文档。
- [ ] 有 GitOps 文档。

### 证据材料

- [ ] 成功 PR。
- [ ] 成功 CI run。
- [ ] 成功 image build run。
- [ ] 镜像 digest。
- [ ] 部署仓库 PR。
- [ ] staging 发布记录。
- [ ] production 发布或模拟记录。
- [ ] 回滚演练记录。
- [ ] 优化报告。
- [ ] 演示脚本。

## 3. 最终面试清单

- [ ] 2 分钟项目介绍。
- [ ] 5 分钟架构讲解。
- [ ] 10 到 15 分钟项目演示。
- [ ] 5 个技术权衡。
- [ ] 5 个排障场景。
- [ ] 3 个安全场景。
- [ ] 3 个优化指标。
- [ ] 1 次模拟面试记录。
- [ ] 简历项目描述。

## 4. 最终自测题

### 题目 1：从 PR 到 production，你的链路是什么？

参考要点：

```text
PR -> CI quality gates -> merge main -> build image -> scan/SBOM/sign -> update deploy repo -> Argo CD staging -> smoke test -> production PR -> approval -> Argo CD production -> observe -> rollback if needed
```

### 题目 2：这个项目里你最重要的 3 个设计决策是什么？

参考要点：

```text
应用仓库和部署仓库分离。
production 使用部署仓库 PR 和 digest。
安全和可观测不是最后补丁，而是发布链路的一部分。
```

### 题目 3：如果 production 发布失败，你怎么处理？

参考要点：

```text
先判断影响范围和回滚信号。
查看发布证据、Argo CD history、Kubernetes events、日志和指标。
如果需要，Git revert 部署仓库或改回旧 digest。
恢复后记录 incident 并复盘根因。
```

### 题目 4：如何证明你的流水线优化有效？

参考要点：

```text
先有优化前基线，例如 p50/p95、最慢 job、失败率。
实施 Go cache、Docker cache、concurrency 或测试分层。
再采集优化后数据对比。
```

### 题目 5：这个项目还有哪些不足？

参考要点：

```text
可以诚实说明演示环境限制，例如 production 是模拟环境、告警未连接真实值班系统、云 OIDC 只做了文档或局部实践。
然后说明下一步计划。
```

## 5. 后续路线

如果你想继续深入，可以选一个方向：

### 平台工程方向

- Backstage。
- 自助发布平台。
- Golden path。
- 内部开发者平台。
- 多服务模板化。

### Kubernetes/云原生方向

- Ingress Gateway。
- Service Mesh。
- HPA/VPA。
- NetworkPolicy。
- Pod Security。
- 多集群 GitOps。

### 安全供应链方向

- SLSA 更高成熟度。
- Admission policy。
- OPA/Gatekeeper/Kyverno。
- Secret 管理平台。
- 合规审计。

### 可观测方向

- OpenTelemetry traces。
- Loki 日志。
- Prometheus 高可用。
- SLO 平台。
- 发布事件和错误预算联动。

### Go 后端方向

- 数据库迁移策略。
- 分布式事务。
- 缓存一致性。
- 性能分析。
- 高并发服务设计。

## 6. 作品集发布建议

如果仓库可以公开：

- 清理所有 secret。
- 使用示例配置。
- 标明哪些是模拟环境。
- 保留成功 workflow run。
- README 写清楚演示方式。

如果仓库不能公开：

- 准备截图。
- 准备脱敏文档。
- 准备 10 分钟口头演示。
- 简历中描述设计和成果，不泄露公司信息。

## 7. 最后一份总结

创建：

```text
portfolio/final-summary.md
```

写：

```markdown
# Final Summary

## What I Built

## What I Learned

## Hard Problems

## Trade-offs

## Evidence

## Next Steps
```

## 8. 本节小结

完成第 10 阶段后，你应该已经具备：

- Go 后端项目工程化能力。
- CI/CD 流程设计能力。
- Docker/Kubernetes/Helm/GitOps 实践能力。
- 安全和供应链意识。
- 发布、回滚和排障能力。
- 可观测和优化意识。
- 面试表达和作品集展示能力。

这条路线到这里就闭环了。

后面真正让你变强的，不是继续堆更多工具名，而是在真实项目中持续使用、复盘和改进这条交付链路。

