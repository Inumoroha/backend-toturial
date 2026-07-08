# 09：Makefile 与本地质量流水线

## 1. 本节目标

前面你已经学了很多命令：

```bash
go fmt ./...
go vet ./...
go test ./...
go test -race ./...
go test -coverprofile=coverage.out ./...
go build ./...
golangci-lint run
govulncheck ./...
```

如果每次都手打，很容易漏。

这一节用 Makefile 把这些命令组织成统一入口。

## 2. 为什么要 Makefile

Makefile 的价值：

- 统一本地命令。
- 降低新成员上手成本。
- 让 CI 复用本地命令。
- 避免文档和实际命令不一致。

目标：

```bash
make ci
```

就能跑完主要质量门禁。

## 3. Windows 用户注意

Windows 默认不一定有 `make`。

你可以选择：

- 使用 Git Bash。
- 使用 WSL。
- 安装 Make。
- 用 PowerShell 手动执行对应命令。

本教程仍然推荐学习 Makefile，因为很多 CI 环境和后端项目都会使用它。

如果暂时没有 `make`，你可以先把命令当作清单逐条执行。

## 4. 基础 Makefile

在项目根目录创建：

```text
Makefile
```

内容：

```makefile
.PHONY: fmt fmt-check vet lint test test-race coverage vuln build ci clean

APP_NAME := go-cicd-lab
BIN_DIR := bin
SERVER_PKG := ./cmd/server

fmt:
	go fmt ./...

fmt-check:
	@test -z "$$(gofmt -l .)" || (gofmt -l . && exit 1)

vet:
	go vet ./...

lint:
	golangci-lint run

test:
	go test ./...

test-race:
	go test -race ./...

coverage:
	go test -coverprofile=coverage.out ./...
	go tool cover -func=coverage.out

vuln:
	govulncheck ./...

build:
	mkdir -p $(BIN_DIR)
	CGO_ENABLED=0 go build -o $(BIN_DIR)/server $(SERVER_PKG)

ci: fmt-check vet lint test test-race coverage vuln build

clean:
	rm -rf $(BIN_DIR) coverage.out
```

## 5. 如果 fmt-check 在 Windows PowerShell 不可用

`fmt-check` 使用了 POSIX shell 语法，适合 Linux、macOS、Git Bash、CI。

Windows PowerShell 可以单独执行：

```powershell
$files = gofmt -l .
if ($files) {
  $files
  exit 1
}
```

后续 CI 通常运行在 Linux runner 上，所以 Makefile 使用 POSIX 写法是常见选择。

## 6. 分层 Make 目标

建议把目标分层：

### 快速检查

```bash
make fmt
make test
```

### PR 前检查

```bash
make vet
make lint
make test
make test-race
make build
```

### 完整检查

```bash
make ci
```

## 7. CI 为什么调用 Makefile

第 3 阶段你会写类似：

```yaml
- run: make ci
```

而不是把所有命令都散落在 YAML 里。

好处：

- 本地和 CI 行为一致。
- 修改检查逻辑只改 Makefile。
- CI 配置更简洁。

## 8. Makefile 失败处理

Makefile 中任何命令返回非 0 状态，目标会失败。

例如：

```makefile
test:
	go test ./...
```

如果测试失败，`make test` 就失败。

CI 会根据退出码判断 job 是否通过。

## 9. 可选：加入集成测试目标

如果你已经有 integration build tag：

```makefile
.PHONY: test-integration

test-integration:
	go test -tags=integration ./...
```

可以让 `ci` 暂时不包含它，先在本地或 main 阶段单独跑。

## 10. 可选：版本构建

```makefile
VERSION ?= dev
COMMIT ?= none
DATE ?= unknown

build:
	mkdir -p $(BIN_DIR)
	CGO_ENABLED=0 go build \
		-ldflags "-X 'github.com/your-name/go-cicd-lab/internal/version.Version=$(VERSION)' -X 'github.com/your-name/go-cicd-lab/internal/version.Commit=$(COMMIT)' -X 'github.com/your-name/go-cicd-lab/internal/version.Date=$(DATE)'" \
		-o $(BIN_DIR)/server $(SERVER_PKG)
```

本地：

```bash
make build VERSION=v0.1.0 COMMIT=$(git rev-parse --short HEAD)
```

CI 中可以把 commit SHA 传进来。

## 11. 小练习

1. 创建 Makefile。
2. 添加目标：

```text
fmt
fmt-check
vet
lint
test
test-race
coverage
vuln
build
ci
clean
```

3. 执行：

```bash
make fmt
make test
make ci
```

如果没有 `make`，逐条执行 Makefile 中对应命令。

## 12. 本节小结

你现在应该理解：

- Makefile 是本地质量流水线的统一入口。
- CI 可以直接复用 Makefile。
- `make ci` 应该代表项目主要质量门禁。
- Windows 本地没有 make 时，可以先手动执行命令或使用 Git Bash/WSL。

