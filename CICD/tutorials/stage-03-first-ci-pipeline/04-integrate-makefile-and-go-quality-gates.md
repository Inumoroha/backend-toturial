# 04：集成 Makefile 与 Go 质量门禁

## 1. 本节目标

第 2 阶段你已经建立了本地质量门禁。

这一节把它们放进 CI：

```text
本地能跑的命令
-> CI 也执行同一套命令
-> PR 失败原因可本地复现
```

## 2. 为什么 CI 要复用 Makefile

如果 CI YAML 里散落很多命令：

```yaml
- run: go vet ./...
- run: go test ./...
- run: go build ./...
```

短期没问题。

但项目变大后，你会遇到：

- 本地文档写一套。
- Makefile 写一套。
- CI YAML 又写一套。
- 三处慢慢不一致。

更好的方式：

```yaml
- run: make ci
```

然后所有检查入口集中在 Makefile。

## 3. 前提：CI runner 需要工具

如果你的 `make ci` 包含：

```text
golangci-lint run
govulncheck ./...
```

那么 CI runner 必须安装：

- `golangci-lint`。
- `govulncheck`。

Go 本身由 `actions/setup-go` 安装。

## 4. 单 job 版本

适合初学阶段，所有检查在一个 job 中顺序执行。

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

jobs:
  quality:
    name: quality
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v6

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.25.x"
          cache: true

      - name: Install govulncheck
        run: go install golang.org/x/vuln/cmd/govulncheck@latest

      - name: Install golangci-lint
        run: go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.0

      - name: Run quality gates
        run: make ci
```

优点：

- 简单。
- 本地和 CI 高度一致。
- 容易理解。

缺点：

- 某一步失败后，后面的检查不跑。
- lint、test、vuln 不能并行。
- 安装工具耗时可能偏长。

## 5. 分 job 版本

当你想让 CI 更快、更清晰时，可以拆 job。

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

jobs:
  test:
    name: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-go@v5
        with:
          go-version: "1.25.x"
          cache: true
      - run: go mod download
      - run: make fmt-check
      - run: make vet
      - run: make test
      - run: make test-race
      - run: make coverage
      - run: make build

  lint:
    name: lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-go@v5
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
      - uses: actions/checkout@v6
      - uses: actions/setup-go@v5
        with:
          go-version: "1.25.x"
          cache: true
      - name: govulncheck
        uses: golang/govulncheck-action@v1
```

优点：

- job 并行运行。
- PR 状态更清楚：test、lint、vuln 谁失败一眼可见。
- `golangci-lint` 使用官方 Action，能得到更好的缓存和注解体验。

缺点：

- YAML 稍长。
- 每个 job 都要 checkout 和 setup Go。

## 6. 推荐初学路线

建议分三步：

1. 先跑最小 CI：`go test`、`go build`。
2. 再改成 `make ci`。
3. 最后拆成 `test`、`lint`、`vuln` 多 job。

不要第一天就追求“完美 CI”。

你需要先让它稳定变绿。

## 7. 让 Makefile 更适合 CI

如果你希望多 job 更干净，可以在 Makefile 中拆目标：

```makefile
.PHONY: ci-core ci lint vuln

ci-core: fmt-check vet test test-race coverage build

ci: ci-core lint vuln
```

这样 CI 可以：

```yaml
- run: make ci-core
```

lint 和 vuln 分别用专门 job 跑。

## 8. 工具版本是否要固定

建议固定关键工具版本。

例如：

```yaml
with:
  version: v2.12.0
```

不建议所有工具都长期使用 `latest`。

原因：

- 新版本可能引入新规则。
- CI 可能突然失败。
- 难以复现历史问题。

学习阶段可以用 `latest` 快速开始，但项目稳定后要逐步固定版本。

## 9. 小练习

完成：

1. 把第一版 CI 改成执行 `make ci`。
2. 如果 CI 找不到 `golangci-lint`，补安装步骤。
3. 如果 CI 找不到 `govulncheck`，补安装步骤。
4. 再尝试把 workflow 拆成 `test`、`lint`、`vuln` 三个 job。
5. 对比两个版本的耗时和可读性。

## 10. 本节小结

你现在应该理解：

- CI 应该尽量复用本地质量门禁。
- `make ci` 是本地和 CI 的桥梁。
- CI runner 需要安装本地没有自带的工具。
- 单 job 简单，多 job 更快且状态更清晰。
- 工具版本应该逐步固定，减少不可预期变化。

