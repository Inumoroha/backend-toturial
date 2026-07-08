# 02：最终要求与评分表

## 1. 本节目标

这一节给毕业项目设置一套验收标准。

你可以把它当作：

```text
自测清单
项目评分表
面试准备路线
```

不要等项目做完才检查，边做边打分。

## 2. 总分结构

建议 100 分：

| 模块 | 分数 |
| --- | ---: |
| Go 服务工程化 | 15 |
| CI 质量门禁 | 15 |
| 镜像与制品 | 10 |
| CD 与 GitOps | 15 |
| 安全与供应链 | 15 |
| 可观测与优化 | 10 |
| 文档与演示 | 10 |
| 面试表达 | 10 |

目标：

```text
70 分：能作为学习项目。
85 分：能作为简历项目。
95 分：能自信做面试演示。
```

## 3. Go 服务工程化：15 分

- [ ] 项目结构清晰：`cmd/`、`internal/`、`pkg/` 或等价结构，2 分。
- [ ] API 可运行，至少有一个核心业务资源，2 分。
- [ ] 有 `/healthz`、`/readyz`、`/version`，2 分。
- [ ] 有 PostgreSQL 或等价持久化，2 分。
- [ ] 有单元测试和集成测试，3 分。
- [ ] 有 Makefile 或脚本统一本地命令，2 分。
- [ ] README 中能说明本地运行方式，2 分。

## 4. CI 质量门禁：15 分

- [ ] PR 自动运行测试，2 分。
- [ ] PR 自动运行 lint，2 分。
- [ ] PR 自动运行 `govulncheck`，2 分。
- [ ] 有覆盖率或测试报告 artifact，2 分。
- [ ] workflow 权限最小化，2 分。
- [ ] 使用缓存优化 Go 依赖，2 分。
- [ ] required checks 或分支保护说明，2 分。
- [ ] 失败日志和 summary 可读，1 分。

## 5. 镜像与制品：10 分

- [ ] Dockerfile 使用多阶段构建，2 分。
- [ ] 运行镜像非 root，2 分。
- [ ] `.dockerignore` 合理，1 分。
- [ ] CI 构建并推送镜像，2 分。
- [ ] 镜像 tag 和 digest 策略清晰，2 分。
- [ ] 镜像构建缓存有优化，1 分。

## 6. CD 与 GitOps：15 分

- [ ] 有部署仓库，2 分。
- [ ] 有 Helm chart，2 分。
- [ ] 有 staging 和 production values，2 分。
- [ ] 有 Argo CD Application，2 分。
- [ ] staging 可自动同步或自动更新，2 分。
- [ ] production 通过 PR 发布，2 分。
- [ ] 有回滚流程，2 分。
- [ ] 有发布审批说明，1 分。

## 7. 安全与供应链：15 分

- [ ] secret 不明文提交，2 分。
- [ ] workflow `permissions` 最小化，2 分。
- [ ] Dependabot 配置，1 分。
- [ ] 镜像扫描，2 分。
- [ ] SBOM 生成，2 分。
- [ ] provenance/attestation，2 分。
- [ ] Cosign 或等价签名，2 分。
- [ ] production secret/environment 保护，1 分。
- [ ] 安全事件 runbook，1 分。

## 8. 可观测与优化：10 分

- [ ] Go 服务暴露 `/metrics`，2 分。
- [ ] 有 smoke test，2 分。
- [ ] 有发布证据链，2 分。
- [ ] 有流水线耗时基线，1 分。
- [ ] 有优化前后报告，2 分。
- [ ] 有 DORA 或交付周报模板，1 分。

## 9. 文档与演示：10 分

- [ ] 应用仓库 README 完整，2 分。
- [ ] 部署仓库 README 完整，2 分。
- [ ] 架构图或流程图，2 分。
- [ ] demo script，2 分。
- [ ] runbook 和 troubleshooting，2 分。

## 10. 面试表达：10 分

- [ ] 能 2 分钟讲清项目背景，2 分。
- [ ] 能 5 分钟讲清 CI/CD 架构，2 分。
- [ ] 能解释 3 个关键权衡，2 分。
- [ ] 能回答失败排查和回滚场景题，2 分。
- [ ] 简历描述具体、有结果，2 分。

## 11. 最低合格线

如果时间有限，至少完成：

```text
Go API + tests
Makefile
Dockerfile
ci.yml
image.yml
deploy repo + Helm chart
staging deployment
production PR 发布设计
README + demo script
```

这已经足够形成一个完整故事。

## 12. 小练习

创建：

```text
docs/final-scorecard.md
```

复制本节评分表，给当前项目打一次分。

然后写：

```markdown
## Top 5 Gaps

1.
2.
3.
4.
5.

## Fix Plan

| Gap | Action | Deadline |
| --- | --- | --- |
```

## 13. 本节小结

你现在有一套清晰评分标准。

后面每完成一个文件，都回到这张表更新分数。

这会帮助你避免“学了很多，但项目无法展示”的尴尬。

