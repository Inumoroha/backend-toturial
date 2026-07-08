# 02：第一条 GitHub Actions Workflow

## 1. 本节目标

这一节创建你的第一条 GitHub Actions workflow。

目标：

```text
PR 到 main 时自动检查。
push 到 main 时自动检查。
检查内容包括依赖下载、测试和构建。
```

## 2. 创建目录

在项目根目录创建：

```text
.github/workflows/
```

命令：

```bash
mkdir -p .github/workflows
```

PowerShell：

```powershell
New-Item -ItemType Directory -Force .github/workflows
```

## 3. 创建 ci.yml

创建文件：

```text
.github/workflows/ci.yml
```

第一版内容：

```yaml
name: ci

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  contents: read

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

      - name: Display Go version
        run: go version

      - name: Download dependencies
        run: go mod download

      - name: Vet
        run: go vet ./...

      - name: Test
        run: go test ./...

      - name: Race test
        run: go test -race ./...

      - name: Build
        run: go build ./...
```

## 4. 每一段是什么意思

### name

```yaml
name: ci
```

workflow 的名字。它会显示在 GitHub Actions 页面和 PR 状态检查中。

### on

```yaml
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
```

触发规则：

- PR 目标分支是 main 时运行。
- push 到 main 时运行。

### permissions

```yaml
permissions:
  contents: read
```

给 `GITHUB_TOKEN` 最小权限。

这条 CI 只需要读取代码，不需要写 issue、不需要推包、不需要部署。

### jobs

```yaml
jobs:
  test:
```

定义一个 job，ID 是 `test`。

### runs-on

```yaml
runs-on: ubuntu-latest
```

使用 GitHub 托管 Ubuntu runner。

### checkout

```yaml
- uses: actions/checkout@v6
```

把仓库代码 checkout 到 runner 上。

没有这一步，runner 上没有你的项目源码。

### setup-go

```yaml
- uses: actions/setup-go@v5
  with:
    go-version: "1.25.x"
    cache: true
```

安装并配置 Go。

`cache: true` 会启用 Go 依赖缓存。GitHub 官方文档说明，`setup-go` 会根据 `go.sum` 等依赖文件参与缓存 key 计算。

## 5. 提交并推送

```bash
git switch -c ci/add-first-workflow
git add .github/workflows/ci.yml
git commit -m "ci: add first GitHub Actions workflow"
git push -u origin ci/add-first-workflow
```

然后创建 PR。

你应该能在 PR 页面看到 `ci / test` 状态。

## 6. 第一次失败很正常

CI 第一次失败很常见。

常见原因：

- `go.mod` 没有提交。
- `go.sum` 没有提交。
- 测试在本地依赖环境变量。
- `go test -race` 暴露了并发问题。
- `go build ./...` 找不到入口包。
- Go 版本和本地不一致。

排查时先看失败的 step，而不是整条 workflow。

## 7. 使用 go-version-file

如果你希望 CI 使用 `go.mod` 中声明的 Go 版本，可以使用：

```yaml
- name: Setup Go
  uses: actions/setup-go@v5
  with:
    go-version-file: go.mod
    cache: true
```

学习阶段两种方式都可以：

- `go-version: "1.25.x"`：明确直观。
- `go-version-file: go.mod`：减少重复配置。

选一种即可，不要同时写两个。

## 8. 小练习

完成：

1. 创建 `.github/workflows/ci.yml`。
2. 提交到 `ci/add-first-workflow` 分支。
3. 创建 PR。
4. 查看 Actions 页面。
5. 故意让一个测试失败，再观察 PR 状态。
6. 修复测试，让 CI 变绿。

## 9. 本节小结

你现在已经有了第一条 GitHub Actions CI。

它完成了：

- 监听 PR 和 main push。
- checkout 代码。
- 安装 Go。
- 下载依赖。
- 运行 vet、test、race、build。
- 把结果反馈到 PR。

