# 01：CI 平台与 Workflow 模型

## 1. 本节目标

这一节先建立 CI 平台的通用模型。

你要知道：

```text
一次 Git 事件
-> 触发 workflow/pipeline
-> CI 平台分配 runner
-> runner 执行 jobs
-> job 执行 steps
-> 产生日志、状态、报告和 artifact
```

后面无论学 GitHub Actions、GitLab CI、Jenkins 还是 Buildkite，底层思路都类似。

## 2. 第 3 阶段只做 CI

本阶段暂时不部署。

目标是：

```text
证明一次代码变更是否具备合并资格。
```

具体检查：

- 代码是否能 checkout。
- Go 环境是否能安装。
- 依赖是否能下载。
- 格式是否正确。
- `go vet` 是否通过。
- 测试是否通过。
- race 检查是否通过。
- lint 是否通过。
- 漏洞检查是否通过。
- 项目是否能 build。

部署 staging、构建 Docker 镜像、发布 production 会在后续阶段学习。

## 3. CI 平台做什么

CI 平台负责：

- 监听 Git 事件。
- 读取仓库中的 CI 配置文件。
- 创建流水线运行记录。
- 分配 runner。
- 注入上下文变量和 secret。
- 执行命令。
- 收集日志和状态。
- 保存 artifact。
- 把结果反馈到 PR 页面。

你写的 YAML 只是配置。真正执行命令的是 runner。

## 4. Runner 是什么

Runner 是执行 CI job 的机器或容器。

它可能是：

- GitHub 托管 runner。
- GitLab shared runner。
- 公司自建 runner。
- Kubernetes 里的临时 Pod。

常见 runner 系统：

```text
ubuntu-latest
windows-latest
macos-latest
```

Go 后端初学阶段推荐：

```text
ubuntu-latest
```

原因：

- 生态最常见。
- 和多数生产 Linux 环境更接近。
- Docker、shell、Makefile 支持更顺滑。
- CI 示例最多。

## 5. Workflow、Job、Step

以 GitHub Actions 为例：

```text
workflow
  job: test
    step: checkout
    step: setup go
    step: go test
  job: lint
    step: checkout
    step: setup go
    step: golangci-lint
```

关系：

```text
workflow 包含多个 job。
job 默认可以并行运行。
job 包含多个 step。
step 顺序执行。
```

如果 job 之间有依赖，需要显式声明。

例如：

```text
lint 和 test 可以并行。
deploy 必须等 test 成功。
```

本阶段主要关注 lint/test/build，不做 deploy。

## 6. CI 的输入和输出

输入：

- Git commit。
- 分支名。
- PR 信息。
- tag。
- workflow 配置。
- secret。
- 依赖缓存。

输出：

- 成功或失败状态。
- job 日志。
- 覆盖率报告。
- 测试报告。
- 构建产物。
- PR 状态检查。

CI 的核心输出不是“日志”，而是一个明确结论：

```text
这次变更可以继续向后走，还是必须停下来修复。
```

## 7. CI 文件放在哪里

GitHub Actions：

```text
.github/workflows/ci.yml
```

GitLab CI：

```text
.gitlab-ci.yml
```

这些配置文件应该提交进 Git。

这样 CI 配置本身也能被 review、追踪和回滚。

## 8. Go 项目的 CI 输入

对 `go-cicd-lab` 来说，CI 至少需要：

```text
go.mod
go.sum
Makefile
.golangci.yml
源码和测试文件
```

如果 `go.sum` 没提交，CI 很可能出现依赖变化。

如果 Makefile 没提交，CI 无法复用本地命令。

## 9. 推荐的第一版 CI

第一版不要复杂。

```text
PR:
  - checkout
  - setup go
  - go mod download
  - go vet ./...
  - go test ./...
  - go test -race ./...
  - go build ./...

push main:
  - 同样检查
```

等这条跑通，再加：

- lint。
- coverage artifact。
- govulncheck。
- matrix。
- branch protection。

## 10. 小练习

回答下面问题：

1. 你的 CI 配置文件应该放在哪里？
2. 你的 CI 应该由哪些 Git 事件触发？
3. 你的第一个 runner 选择什么系统？
4. 你的第一版 CI 至少跑哪些 Go 命令？
5. 哪些命令来自第 2 阶段的本地质量门禁？

## 11. 本节小结

你现在应该理解：

- CI 平台监听 Git 事件并创建 workflow/pipeline。
- runner 执行 job，job 执行 step。
- 第 3 阶段只关注代码检查，不做部署。
- CI 配置应该提交进 Git，并通过 PR review。
- 第一版 CI 越简单越容易跑通。

