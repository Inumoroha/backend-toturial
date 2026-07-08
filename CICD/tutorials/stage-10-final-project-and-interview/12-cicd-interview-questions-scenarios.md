# 12：CI/CD 场景面试题

## 1. 本节目标

场景题考察的是排障和设计能力。

这一节整理常见 CI/CD 场景题，并给出回答思路。

## 2. 场景 1：PR CI 突然变慢

问题：

```text
最近 PR CI 从 6 分钟变成 20 分钟，你怎么排查？
```

回答思路：

```text
先看最近 workflow runs 的 p50/p95，确认是普遍变慢还是个别长尾。
再看 job 耗时，拆分 setup、依赖下载、测试、Docker build、扫描。
检查最近是否改过 go.sum、Dockerfile、workflow、runner 类型或 matrix。
看 cache 是否 miss。
最后基于瓶颈提出优化，例如 Go cache、buildx cache、测试分层或 concurrency。
```

## 3. 场景 2：main CI 失败但 PR 通过

回答思路：

```text
比较 PR workflow 和 main workflow 的触发条件、权限、环境变量和执行命令。
main 可能多跑了镜像构建、安全扫描、部署更新或 integration test。
检查是否只有 main 能访问某些 secret 或 packages 权限。
用 run id 找到失败 step，再看最近合并的 PR。
```

## 4. 场景 3：Docker 镜像本地能跑，Kubernetes 不健康

回答思路：

```text
先看 Pod 状态、events、describe、logs。
检查 readiness/liveness probe 路径和端口。
检查容器监听地址是否是 0.0.0.0。
检查环境变量、ConfigMap、Secret。
检查资源限制、数据库连接和镜像架构。
如果 Argo CD Synced 但服务不可用，继续查运行时状态。
```

## 5. 场景 4：production 发布后接口 500

回答思路：

```text
先确认影响范围和是否需要回滚。
查看发布证据：deploy PR、commit、image digest、workflow run、Argo CD history。
看 /version 确认当前运行版本。
看日志、错误率、trace、Kubernetes events。
如果符合回滚信号，先 Git revert 部署仓库或改回旧 digest。
恢复后再复盘根因。
```

## 6. 场景 5：secret 泄露到 Git

回答思路：

```text
第一步不是删历史，而是撤销或轮换 secret。
然后检查影响范围：Git 历史、workflow logs、artifact、镜像层、访问日志。
重新发布干净制品。
清理历史和缓存。
最后补 secret scanning、review 规则和 runbook。
```

## 7. 场景 6：镜像扫描发现 CRITICAL 漏洞

回答思路：

```text
先确认漏洞来源：基础镜像、系统包、Go module 或工具层。
看是否有修复版本。
如果是 production 发布，应按策略阻断或安全审批。
如果无修复版本，要记录风险接受、影响范围和缓解措施。
不能简单关闭扫描。
```

## 8. 场景 7：Argo CD OutOfSync

回答思路：

```text
先看 diff，确认是 Git 变了未同步，还是集群被手动改了。
如果是正常发布，sync 即可。
如果是 drift，要判断手动修改是否合理。
合理变更补回 Git，不合理变更让 Argo CD 恢复期望状态。
production 要注意 prune/selfHeal 风险。
```

## 9. 场景 8：镜像 tag 被覆盖

回答思路：

```text
解释 tag 可以移动，所以 production 应记录 digest。
排查 registry push 记录和 workflow run。
确认当前部署 digest 是否可信。
后续改进：immutable tag、digest 发布、签名验证、部署仓库记录 digest。
```

## 10. 场景 9：flaky test 频繁失败

回答思路：

```text
先统计失败的 test name、package、run id 和重跑结果。
分类是时间依赖、并发竞态、共享状态还是外部服务不稳定。
短期可以隔离或重试，但必须建 issue 指定 owner。
长期修复测试隔离、mock、随机端口、测试数据和超时策略。
```

## 11. 场景 10：如何设计从零到一的 CI/CD

回答结构：

```text
1. 先建立本地命令：test/lint/build。
2. PR CI 跑快速质量门禁。
3. main 构建镜像并推送 registry。
4. staging 自动部署和 smoke test。
5. production 加审批、PR、回滚。
6. 加安全扫描、SBOM、签名。
7. 加观测和优化。
```

强调：

```text
先可靠，再复杂。
```

## 12. 场景 11：如何处理数据库迁移

回答思路：

```text
数据库迁移要和应用发布兼容。
优先使用向前兼容迁移，例如先加字段再切代码，最后清理旧字段。
迁移要有备份、回滚或补救策略。
CI 可以检查 migration 格式，staging 先验证。
production 发布前要评估锁表、耗时和回滚风险。
```

## 13. 场景 12：如何做多服务 CI/CD

回答思路：

```text
先明确 monorepo 还是 multi-repo。
使用路径过滤减少无关服务构建。
每个服务有独立镜像和部署配置。
共享模板要控制版本。
跨服务发布要有契约测试和兼容策略。
```

## 14. 小练习

选择 5 个场景题，用下面格式回答：

```markdown
## Scenario

## First Response

## Investigation

## Fix

## Prevention

## Example from go-cicd-lab
```

## 15. 本节小结

场景题要体现：

- 先止血。
- 再定位。
- 再修复。
- 最后预防。

不要只回答工具命令，要回答决策顺序。

