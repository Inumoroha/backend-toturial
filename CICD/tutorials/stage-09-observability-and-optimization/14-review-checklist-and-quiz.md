# 14：复习清单与自测题

## 1. 本节目标

用清单和问题确认你是否掌握第 9 阶段。

如果你能用数据分析流水线、优化缓存和并发、治理 flaky test、记录发布证据，并用 DORA 指标描述交付效率，你已经具备比较完整的 Go 后端 CI/CD 工程化能力。

## 2. 操作清单

### 可观测性基础

- [ ] 我能解释 CI/CD 中 logs、metrics、events、artifacts 的区别。
- [ ] 我能画出从 PR 到 production 的可观测性地图。
- [ ] 我知道 PR 反馈、发布反馈、运行时反馈分别看什么。

### Workflow 排查

- [ ] 我能用日志分组提升可读性。
- [ ] 我能用 `$GITHUB_STEP_SUMMARY` 输出摘要。
- [ ] 我能失败时上传测试报告 artifact。
- [ ] 我能用 `gh run list` 和 `gh run view --log-failed` 排查失败。

### 指标采集

- [ ] 我能导出最近 workflow runs。
- [ ] 我能统计 success/failure/cancelled。
- [ ] 我能查看某个 run 的 jobs。
- [ ] 我能记录 CI p50/p95。
- [ ] 我知道 cancelled 不一定等同 failure。

### 优化基线

- [ ] 我能建立优化前基线。
- [ ] 我能拆分 setup、test、build、scan、deploy 耗时。
- [ ] 我能写出优化假设。
- [ ] 我能比较优化前后指标。

### 缓存与构建

- [ ] 我能启用 `setup-go cache: true`。
- [ ] 我能解释 cache key 和 restore key。
- [ ] 我知道缓存不能包含 secret。
- [ ] 我能优化 Dockerfile 层顺序。
- [ ] 我能使用 buildx `cache-from/cache-to`。
- [ ] 我能用 `.dockerignore` 减少构建上下文。

### 并行与并发

- [ ] 我能拆分 test、lint、vuln job。
- [ ] 我能用 `needs` 表达依赖。
- [ ] 我能使用 matrix。
- [ ] 我能用 `concurrency` 取消过时 PR run。
- [ ] 我知道 production deploy 应串行。

### 测试治理

- [ ] 我能区分 unit、integration、e2e、smoke test。
- [ ] 我能用 `go test -short`。
- [ ] 我能保存 `go test -json`。
- [ ] 我能分类失败原因。
- [ ] 我能写 flaky test 治理流程。

### 发布与运行时观测

- [ ] 我能记录 commit、run URL、image digest、deploy repo commit。
- [ ] 我能执行 smoke test。
- [ ] 我能查看 Argo CD app history。
- [ ] 我能给 Go 服务增加 `/metrics`。
- [ ] 我能解释 label cardinality。
- [ ] 我能写请求量、错误率、p95 延迟的 PromQL。

### Dashboard、告警与 DORA

- [ ] 我能设计 CI/CD dashboard。
- [ ] 我能设计发布观察 dashboard。
- [ ] 我能写一个高错误率告警。
- [ ] 我能解释 SLO 和错误预算。
- [ ] 我能解释 DORA 四项指标。
- [ ] 我知道 DORA 不应作为个人考核武器。
- [ ] 我能写交付周报。

## 3. 自测题

### 题目 1：为什么优化前必须建立基线？

参考答案：

```text
没有基线就无法证明优化是否有效。
团队容易凭感觉做改动，可能优化了并不慢的地方，或者引入复杂度却没有收益。
基线应包括 p50/p95、成功率、失败热点和阶段耗时。
```

### 题目 2：日志和指标分别解决什么问题？

参考答案：

```text
日志适合排查一次具体失败，告诉你发生了什么。
指标适合看趋势和对比，告诉你是否变慢、失败率是否升高、优化是否有效。
两者需要结合使用。
```

### 题目 3：为什么 PR workflow 适合使用 `cancel-in-progress: true`？

参考答案：

```text
同一个 PR 连续 push 后，旧 commit 的结果通常已经没有价值。
取消旧 run 可以节省 runner 时间，让开发者关注最新 commit 的反馈。
但 production deploy 不适合随便取消，应使用固定 concurrency group 串行执行。
```

### 题目 4：为什么 Dockerfile 中要先复制 `go.mod` 和 `go.sum`？

参考答案：

```text
Docker 按层缓存。
先复制 go.mod/go.sum 并执行 go mod download，可以在普通源码变化时复用依赖下载层。
如果先 COPY . .，任何源码变化都会让依赖下载层失效。
```

### 题目 5：flaky test 为什么危险？

参考答案：

```text
flaky test 会让开发者不再相信 CI。
当失败可以通过 rerun 解决时，团队会逐渐忽视真实问题。
flaky test 应记录、分类、指定 owner、限期修复，重试只能临时缓解。
```

### 题目 6：发布后为什么还要 smoke test？

参考答案：

```text
CI 通过只说明代码检查通过。
部署成功只说明资源更新完成。
smoke test 验证服务发布后是否具备最小可用能力，例如 health、ready、version 和关键 API。
```

### 题目 7：Prometheus label cardinality 为什么重要？

参考答案：

```text
高 cardinality 会产生大量时间序列，增加存储和查询压力。
不应把 user id、request id、原始 URL、错误全文作为 label。
路由应使用模板形式，例如 /users/{id}。
```

### 题目 8：DORA 四项指标是什么？

参考答案：

```text
部署频率、变更前置时间、变更失败率、服务恢复时间。
它们用于描述团队交付系统的速度和稳定性，不应用来简单考核个人。
```

## 4. 情景题

### 情景 1

团队说“CI 太慢”，你会先做什么？

参考思路：

```text
先导出最近 workflow runs，记录 p50/p95、成功率和最慢 job。
再拆分 setup、test、build、scan、deploy 的耗时。
基于数据提出优化假设，而不是直接改 YAML。
```

### 情景 2

你开启了 Docker build cache，但构建速度没有变快。

你会查什么？

参考思路：

```text
检查 Dockerfile 层顺序。
检查 .dockerignore。
检查 buildx 是否设置 cache-from/cache-to。
检查源码变化是否导致早期层失效。
检查是否每次都是多平台构建。
检查日志中 cache 是否命中。
```

### 情景 3

production 发布后 workflow 成功，但用户报告接口 500。

你会查什么？

参考思路：

```text
查看发布证据：commit、run URL、image digest、deploy repo commit。
查看 Argo CD history 和 Kubernetes events。
查看 /version 确认运行版本。
查看错误率、日志、trace 和最近部署时间是否相关。
必要时按预定义回滚信号回滚。
```

### 情景 4

团队想把所有慢测试从 PR 中删掉。

你怎么判断？

参考思路：

```text
先区分 unit、integration、e2e 和 smoke test。
PR 可以保留快速高价值测试，慢集成测试移到 main/nightly。
不能直接删除覆盖关键风险的测试。
要记录分层策略和失败反馈位置。
```

## 5. 第 9 阶段最终作业

为 `go-cicd-lab` 输出：

```text
docs/cicd-observability.md
docs/pipeline-optimization-report.md
docs/delivery-report-template.md
scripts/smoke-test.sh
.github/workflows/ci.yml
.github/workflows/image.yml
```

要求：

- 能导出最近 workflow runs。
- 能统计 CI 成功率和 p50/p95。
- 能指出最慢 job。
- `ci.yml` 输出 job summary。
- 测试失败时上传报告 artifact。
- Go cache 已启用。
- Docker build cache 已启用。
- PR workflow 使用 concurrency 取消旧 run。
- production deploy 串行。
- Go 服务有 `/metrics` 和 `/version`。
- 发布后 smoke test 能运行。
- 有一份优化前后对比报告。

## 6. 阶段完成标准

当你能做到下面几件事，就完成了第 9 阶段：

1. 能用数据说明流水线哪里慢。
2. 能建立优化前后基线。
3. 能优化 Go 和 Docker 缓存。
4. 能合理使用并行、matrix 和 concurrency。
5. 能治理 flaky test。
6. 能为发布建立证据链和 smoke test。
7. 能给 Go 服务暴露 metrics。
8. 能设计 dashboard 和基础告警。
9. 能解释 DORA 四项指标。
10. 能写交付周报和优化报告。

完成到这里，你已经不只是“会写 CI/CD 配置”，而是能持续经营一条 Go 后端交付链路。

