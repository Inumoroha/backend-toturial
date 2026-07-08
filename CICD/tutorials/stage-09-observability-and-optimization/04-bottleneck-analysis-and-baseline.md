# 04：瓶颈分析与优化基线

## 1. 本节目标

优化流水线之前，先回答：

```text
到底慢在哪里？
优化目标是什么？
优化后如何证明有效？
```

这一节学习如何拆解耗时、建立基线、制定优化假设。

## 2. 一条 CI 的典型耗时

Go 后端 CI 常见阶段：

```text
queue
checkout
setup-go
go mod download
go test
lint
govulncheck
docker build
image scan
artifact upload
```

CD 阶段还包括：

```text
approval wait
image pull
helm template
argocd sync
rollout
smoke test
```

不要只看总耗时。

总耗时是结果，阶段耗时才是线索。

## 3. 建立基线表

创建：

```text
docs/pipeline-optimization-report.md
```

写入：

```markdown
# Pipeline Optimization Report

## Baseline

Date: YYYY-MM-DD
Repository:
Workflow:

| Metric | Value |
| --- | --- |
| PR CI p50 | TBD |
| PR CI p95 | TBD |
| CI success rate | TBD |
| Slowest job | TBD |
| Docker build average | TBD |
| Go test average | TBD |
| Image scan average | TBD |

## Top Bottlenecks

1. TBD
2. TBD
3. TBD

## Optimization Hypotheses

| Hypothesis | Expected Impact | How to Verify |
| --- | --- | --- |
| Enable Go cache | Reduce setup/test time | Compare 20 runs before/after |
```

## 4. 分析 p50 和 p95

平均值容易被极端值影响。

建议至少看：

```text
p50：多数开发者感受到的普通速度。
p95：慢的时候有多痛。
max：最坏情况。
```

例子：

| 指标 | 数值 |
| --- | --- |
| p50 | 6 分钟 |
| p95 | 22 分钟 |
| max | 40 分钟 |

这说明：

```text
大多数时候还可以，但存在严重长尾。
```

可能原因：

- runner 排队。
- 依赖下载不稳定。
- Docker cache 偶尔失效。
- 某些集成测试很慢。
- 外部服务不稳定。

## 5. 区分可优化和不可优化

可优化：

- 无效重复安装依赖。
- Dockerfile 层顺序差。
- 所有测试串行运行。
- PR 每次都跑完整 e2e。
- 多平台镜像在每个 PR 都构建。
- 旧 run 不取消，浪费 runner。

不一定可优化：

- 安全扫描本身需要时间。
- production 审批等待。
- 必要的集成测试。
- 云资源限速。

可优化不等于应该立刻优化。

先看收益：

```text
优化 20 秒是否值得增加 100 行复杂 YAML？
```

## 6. 用假设驱动优化

坏做法：

```text
我们把所有 job 都并行吧。
```

好做法：

```text
假设：Go module 下载耗时占 PR CI 30%。
方案：启用 setup-go cache，并固定 cache-dependency-path。
验证：比较修改前后最近 20 次 PR CI 的 go mod download 时间。
```

每个优化项都写：

- 假设。
- 改动。
- 预期收益。
- 风险。
- 验证方式。

## 7. 常见瓶颈判断

### setup 慢

表现：

```text
checkout/setup-go/install tool 占比高。
```

可能优化：

- 使用官方 setup action 缓存。
- 减少每次安装工具。
- 将工具版本固定。
- 使用预装 runner 或自托管 runner。

### dependency 慢

表现：

```text
go mod download 时间长。
```

可能优化：

- 启用 Go module cache。
- 使用内部 proxy。
- 检查依赖体积。

### test 慢

表现：

```text
go test 占比高。
```

可能优化：

- 分层测试。
- 并行 package。
- 慢测试移到 nightly 或集成测试 workflow。
- 减少不必要外部依赖。

### Docker build 慢

表现：

```text
每次都重新下载依赖和编译。
```

可能优化：

- 调整 Dockerfile COPY 顺序。
- 使用 buildx cache。
- 使用 `.dockerignore`。

### deploy 慢

表现：

```text
rollout 或 smoke test 超时。
```

可能优化：

- 优化 readinessProbe。
- 检查镜像拉取速度。
- 缩短无效等待。
- 增加部署事件日志。

## 8. 小练习

选择最近 10 次 PR CI，填写：

```markdown
| Run | Total | setup | test | build | scan | deploy | Result |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | | | | | | | |
```

然后写出：

1. 最慢阶段是什么？
2. 最常失败阶段是什么？
3. 第一个优化假设是什么？
4. 如何验证这个假设？

## 9. 本节小结

你现在应该理解：

- 优化前先建立基线。
- p50 和 p95 比单纯平均值更有解释力。
- 阶段耗时比总耗时更能定位问题。
- 每个优化都应有假设和验证方式。
- 不值得为了很小收益引入过大复杂度。

