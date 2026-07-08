# 08：实战，为 go-cicd-lab 建立第一条 CI

## 1. 本节目标

这一节给 `go-cicd-lab` 落地一条完整但不过度复杂的 CI。

最终你要得到：

```text
.github/workflows/ci.yml
docs/ci.md
```

并让 PR 页面显示：

```text
ci / test
ci / lint
ci / vuln
```

## 2. 前置检查

确认项目根目录有：

```text
go.mod
go.sum
Makefile
.golangci.yml
```

本地先跑：

```bash
go mod tidy
go test ./...
go test -race ./...
go build ./...
```

如果这些本地都失败，先不要写 CI。

## 3. 推荐 Makefile 调整

如果你的 Makefile 还没有 `ci-core`，建议加上：

```makefile
.PHONY: ci-core

ci-core: fmt-check vet test test-race coverage build
```

保留：

```makefile
ci: ci-core lint vuln
```

这样 GitHub Actions 可以：

- `test` job 跑 `make ci-core`。
- `lint` job 使用 `golangci-lint-action`。
- `vuln` job 使用 `govulncheck-action`。

## 4. 创建 workflow

创建：

```text
.github/workflows/ci.yml
```

内容：

```yaml
name: ci

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    name: test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v6

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.25.x"
          cache: true

      - name: Download dependencies
        run: go mod download

      - name: Run core quality gates
        run: make ci-core

      - name: Upload coverage
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage.out
          if-no-files-found: ignore

      - name: Coverage summary
        if: always()
        run: |
          if [ -f coverage.out ]; then
            echo "## Coverage" >> "$GITHUB_STEP_SUMMARY"
            go tool cover -func=coverage.out | tail -n 1 >> "$GITHUB_STEP_SUMMARY"
          fi

  lint:
    name: lint
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v6

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.25.x"
          cache: true

      - name: golangci-lint
        uses: golangci/golangci-lint-action@v9
        with:
          version: v2.12.0

  vuln:
    name: vuln
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v6

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.25.x"
          cache: true

      - name: govulncheck
        uses: golang/govulncheck-action@v1
```

## 5. concurrency 是什么

```yaml
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

含义：

```text
同一个分支上新的 CI 运行开始时，取消还在跑的旧 CI。
```

适合 PR 场景：

- 你连续 push 了多次。
- 旧 commit 的 CI 已经没意义。
- 取消旧运行可以节省时间。

## 6. 为什么拆三个 job

```text
test: fmt/vet/test/race/coverage/build
lint: golangci-lint
vuln: govulncheck
```

优点：

- 并行更快。
- 状态更清楚。
- lint 使用官方 Action。
- vuln 使用 Go 官方 Action。

如果你只想简单，也可以先用一个 job 跑 `make ci`。

## 7. 提交 workflow

```bash
git switch -c ci/add-go-quality-workflow
git add .github/workflows/ci.yml Makefile
git commit -m "ci: add Go quality workflow"
git push -u origin ci/add-go-quality-workflow
```

创建 PR。

观察：

- Actions 页面是否出现运行记录。
- PR 是否显示 `test`、`lint`、`vuln`。
- 失败时是哪一个 job。

## 8. 配置 required checks

CI 稳定后，在 GitHub 仓库设置里配置 main 分支保护：

```text
Require status checks to pass before merging
```

选择：

```text
ci / test
ci / lint
ci / vuln
```

这样 CI 失败时不能合并 main。

## 9. 创建 docs/ci.md

创建：

```text
docs/ci.md
```

内容模板：

````markdown
# CI 说明

## 触发规则

- PR 到 main：运行 Go CI。
- push main：运行 Go CI。
- workflow_dispatch：允许手动运行。

## Jobs

- `test`：fmt-check、vet、test、race、coverage、build。
- `lint`：golangci-lint。
- `vuln`：govulncheck。

## 本地复现

```bash
make ci-core
make lint
make vuln
```

## Artifact

- `coverage`：覆盖率文件 `coverage.out`。

## 分支保护

main 分支要求以下检查通过：

- `ci / test`
- `ci / lint`
- `ci / vuln`
````

如果你要把这段模板放进 Markdown 文件，外层代码围栏需要用四个反引号，避免嵌套冲突。

## 10. 本节验收

你完成后应该能做到：

- PR 自动触发 CI。
- main push 自动触发 CI。
- Actions 页面能看到 `test`、`lint`、`vuln`。
- 覆盖率 artifact 能下载。
- main 分支配置 required checks。
- 本地能复现 CI 中的大多数命令。

## 11. 本节小结

你现在已经给 `go-cicd-lab` 建立了第一条可用 CI。

它还没有构建 Docker 镜像，也没有部署环境，但已经能承担 PR 质量门禁职责。
