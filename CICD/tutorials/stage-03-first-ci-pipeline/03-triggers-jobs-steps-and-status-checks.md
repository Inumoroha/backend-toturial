# 03：触发器、Job、Step 与状态检查

## 1. 本节目标

上一节你写了第一条 workflow。

这一节深入理解：

- `on` 如何触发 workflow。
- job 默认如何运行。
- step 失败会发生什么。
- required status checks 如何保护 main。

## 2. 常见触发器

### pull_request

```yaml
on:
  pull_request:
    branches: [main]
```

当有人向 main 创建或更新 PR 时运行。

适合：

- lint。
- test。
- build check。
- 基础安全检查。

不适合：

- 使用生产 secret。
- 部署 production。
- 执行危险写操作。

### push main

```yaml
on:
  push:
    branches: [main]
```

当代码进入 main 后运行。

适合：

- 再跑一次完整检查。
- 构建制品。
- 后续阶段部署 staging。

第 3 阶段先只做检查。

### tag

```yaml
on:
  push:
    tags:
      - "v*"
```

当推送版本 tag 时运行。

适合后续阶段：

- 创建 release。
- 构建正式镜像。
- 发布 production。

第 3 阶段先理解，不急着用。

### workflow_dispatch

```yaml
on:
  workflow_dispatch:
```

允许手动运行 workflow。

适合：

- 手动补跑。
- 手动安全扫描。
- 后续手动部署或回滚。

## 3. 多触发器组合

常见写法：

```yaml
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
  workflow_dispatch:
```

这表示：

- PR 到 main 跑。
- push main 跑。
- 也允许手动跑。

## 4. 路径过滤

如果只改文档，是否要跑完整 Go CI？

可以用 paths：

```yaml
on:
  pull_request:
    branches: [main]
    paths:
      - "**.go"
      - "go.mod"
      - "go.sum"
      - "Makefile"
      - ".github/workflows/**"
```

初学阶段不建议过早加路径过滤。

原因：

- 容易漏掉影响构建的文件。
- 规则复杂后不好排查。

等项目大了、CI 很慢时再优化。

## 5. Job 默认并行

多个 job 默认并行运行：

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: go test ./...

  lint:
    runs-on: ubuntu-latest
    steps:
      - run: golangci-lint run
```

并行能节省时间，但每个 job 都是独立 runner 环境。

所以每个 job 通常都需要：

```yaml
- uses: actions/checkout@v6
- uses: actions/setup-go@v5
```

## 6. 用 needs 控制顺序

如果一个 job 必须等另一个 job 成功：

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: go test ./...

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - run: go build ./...
```

`build` 会等 `test` 成功后才运行。

第 3 阶段的 CI 多数检查可以并行，但初学阶段可以先写一个 job，简单可靠。

## 7. Step 失败会发生什么

一个 step 返回非 0 状态，默认会让当前 job 失败。

例如：

```yaml
- name: Test
  run: go test ./...
```

如果测试失败，后续 step 默认不会继续执行，job 会失败，workflow 也会显示失败。

这就是质量门禁的基础。

## 8. continue-on-error

可以让某个 step 失败但不阻断：

```yaml
- name: Vulnerability check
  run: govulncheck ./...
  continue-on-error: true
```

初学阶段谨慎使用。

建议：

- 核心测试不要 `continue-on-error`。
- lint 不建议忽略。
- 安全扫描是否阻断，需要团队规则。

## 9. Required Status Checks

CI 跑起来之后，要让它成为 main 的保护门禁。

在 GitHub 仓库设置中配置：

```text
Settings
-> Branches
-> Branch protection rules
-> main
-> Require status checks to pass before merging
```

选择你的检查，例如：

```text
ci / test
```

这样 PR 必须等 CI 通过才能合并。

## 10. 状态检查命名

PR 上显示的名字通常由 workflow name 和 job name 组成。

例如：

```yaml
name: ci

jobs:
  test:
    name: test
```

PR 上可能显示：

```text
ci / test
```

如果你改了 job name，required check 名称也可能需要更新。

所以不要频繁改 job 名字。

## 11. 小练习

完成：

1. 给 `ci.yml` 加上 `workflow_dispatch`。
2. 在 GitHub Actions 页面手动运行一次。
3. 给 main 配置 required status checks。
4. 创建一个故意失败的 PR，观察是否被阻止合并。
5. 修复后再观察状态变化。

## 12. 本节小结

你现在应该理解：

- `pull_request` 适合 PR 检查。
- `push main` 适合主干检查。
- `workflow_dispatch` 适合手动补跑。
- job 默认并行，`needs` 控制依赖。
- step 失败会让 job 失败。
- required status checks 能把 CI 接到 main 分支保护上。

