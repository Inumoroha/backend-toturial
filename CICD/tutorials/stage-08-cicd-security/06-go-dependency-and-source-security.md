# 06：Go 依赖与源码安全

## 1. 本节目标

CI/CD 安全不只发生在部署阶段。

Go 服务的依赖、源码、生成代码、私有模块访问方式，都会影响最终制品的可信度。

这一节学习：

- 如何检查 Go module 依赖。
- 如何使用 `govulncheck`。
- 如何配置 Dependabot。
- 如何理解 CodeQL/代码扫描。
- 如何把源码安全纳入 PR 流程。

## 2. Go 依赖安全的核心问题

Go 项目通常依赖：

```text
go.mod
go.sum
直接依赖
间接依赖
私有模块
生成代码
工具依赖
```

风险包括：

- 依赖库存在已知漏洞。
- 间接依赖被引入高危漏洞。
- `go.mod` 被异常修改。
- `go.sum` 被恶意替换。
- 私有模块访问 token 权限过大。
- CI 下载依赖时泄露凭证。

PR 中看到下面文件变化时要认真审查：

```text
go.mod
go.sum
tools.go
Makefile
.github/workflows/*.yml
Dockerfile
```

## 3. 基础命令：整理与查看依赖

在项目根目录执行：

```bash
go mod tidy
```

作用：

- 删除未使用依赖。
- 增加缺失依赖。
- 更新 `go.mod` 和 `go.sum` 到一致状态。

查看所有模块：

```bash
go list -m all
```

查看可升级模块：

```bash
go list -u -m all
```

只查看主模块的直接依赖：

```bash
go list -m -json all
```

如果依赖突然大量变化，不要只看 CI 是否通过，还要问：

```text
这些依赖为什么被引入？
是否来自可信维护者？
是否有更小的替代方案？
是否真的属于当前需求？
```

## 4. 使用 govulncheck

`govulncheck` 会结合 Go 漏洞数据库和代码调用路径，检查项目是否实际调用了存在漏洞的符号。

安装：

```bash
go install golang.org/x/vuln/cmd/govulncheck@latest
```

本地扫描：

```bash
govulncheck ./...
```

建议加入 `Makefile`：

```makefile
.PHONY: vuln
vuln:
	govulncheck ./...
```

CI 中调用：

```yaml
name: ci

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true

      - name: Install govulncheck
        run: go install golang.org/x/vuln/cmd/govulncheck@latest

      - name: Vulnerability check
        run: govulncheck ./...
```

学习阶段可以先让它阻断 PR。

生产中可以按团队成熟度拆分策略：

```text
高危漏洞阻断合并。
中低危漏洞创建 issue 并限期处理。
无可用修复版本时记录风险接受。
```

## 5. Dependabot：自动发现依赖更新

创建：

```text
.github/dependabot.yml
```

示例：

```yaml
version: 2
updates:
  - package-ecosystem: gomod
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 5
    groups:
      go-minor-and-patch:
        update-types:
          - minor
          - patch

  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 5
```

作用：

- 自动创建依赖更新 PR。
- 自动创建 GitHub Actions 版本更新 PR。
- 让依赖升级进入正常 review 流程。

注意：

```text
Dependabot 不是自动合并工具。
它负责提醒和创建 PR，是否合并仍要看测试、安全扫描和评审。
```

## 6. CodeQL 与代码扫描

`govulncheck` 主要关注 Go 依赖漏洞和调用路径。

CodeQL/代码扫描更偏向源码模式问题，例如：

- 注入风险。
- 不安全的路径处理。
- 不安全的加密使用。
- 反序列化风险。
- 部分命令执行风险。

示例：

```yaml
name: codeql

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 2 * * 1"

permissions:
  contents: read
  security-events: write

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: github/codeql-action/init@v3
        with:
          languages: go

      - uses: github/codeql-action/analyze@v3
```

如果你使用的是 GitLab、SonarQube 或其他平台，思路一样：

```text
把静态分析结果变成 PR 可见的反馈。
让严重问题不能悄悄进入主分支。
```

## 7. 私有 Go Module 的凭证边界

私有模块常见配置：

```bash
go env -w GOPRIVATE=github.com/your-org/*
```

CI 中不要使用个人长期 PAT 作为万能钥匙。

更好的方向：

- 使用只读 deploy key。
- 使用 GitHub App token。
- 使用细粒度 PAT，且只授予所需仓库只读权限。
- 定期轮换。
- 不在日志中输出 token。

错误示例：

```yaml
- run: git config --global url."https://${{ secrets.PAT }}@github.com/".insteadOf "https://github.com/"
```

这类写法容易扩大 token 暴露面。

更安全的原则：

```text
只让需要下载私有依赖的 job 拿到凭证。
凭证只读。
凭证不进入镜像层。
凭证不进入构建 artifact。
```

## 8. PR 审查重点

当 PR 修改依赖时，review 重点：

- 新增依赖是否必要。
- 是否引入重量级框架。
- 是否有长期维护。
- 是否有已知漏洞。
- 是否修改了 `go.sum` 中大量无关条目。
- 是否修改了构建脚本。
- 是否修改了 workflow 权限。

可以在 PR 模板加入：

```markdown
## Dependency changes

- [ ] This PR changes `go.mod` or `go.sum`.
- [ ] New dependencies are necessary for this feature.
- [ ] I checked known vulnerabilities.
- [ ] CI passed `govulncheck ./...`.
```

## 9. 推荐流水线顺序

对 Go 服务，建议 PR 阶段至少包含：

```text
checkout
setup-go
go mod download
go test ./...
golangci-lint
govulncheck ./...
CodeQL 或平台代码扫描
```

构建镜像前先通过源码和依赖检查。

原因很简单：

```text
不要把已经知道有问题的源码继续打包成镜像。
```

## 10. 小练习

在你的 `go-cicd-lab` 中完成：

1. 运行 `go mod tidy`。
2. 运行 `go list -u -m all`。
3. 安装并运行 `govulncheck ./...`。
4. 新增 `.github/dependabot.yml`。
5. 新增或检查 CodeQL workflow。
6. 在 PR 模板中加入依赖变更检查项。

## 11. 本节小结

你现在应该理解：

- Go 依赖安全要从 `go.mod` 和 `go.sum` 开始。
- `govulncheck` 应成为 Go 项目基础质量门禁。
- Dependabot 负责提醒更新，不等于自动安全。
- CodeQL/代码扫描补充源码层面的安全检查。
- 私有模块凭证要只读、最小权限、只在必要 job 使用。

