# 10：实战，为 go-cicd-lab 建立质量门禁

## 1. 本节目标

这一节把第 2 阶段内容落到 `go-cicd-lab`。

最终你要得到：

- Go module。
- 基础项目结构。
- 单元测试。
- HTTP handler 测试。
- `.golangci.yml`。
- `.gitignore`。
- `.env.example`。
- Makefile。
- 一条本地可运行的 `make ci`。

## 2. 初始化项目

如果你还没有 Go module：

```bash
go mod init github.com/your-name/go-cicd-lab
```

创建目录：

```bash
mkdir -p cmd/server internal/handler internal/service internal/version docs
```

PowerShell：

```powershell
New-Item -ItemType Directory -Force cmd/server,internal/handler,internal/service,internal/version,docs
```

## 3. 添加服务入口

创建：

```text
cmd/server/main.go
```

内容：

```go
package main

import "fmt"

func main() {
	fmt.Println("go-cicd-lab")
}
```

后面阶段会把它替换成真实 HTTP 服务。

## 4. 添加 service 代码

创建：

```text
internal/service/todo.go
```

内容：

```go
package service

import (
	"errors"
	"strings"
)

var ErrEmptyTitle = errors.New("empty title")

func NormalizeTitle(title string) string {
	return strings.TrimSpace(title)
}

func ValidateTitle(title string) error {
	if NormalizeTitle(title) == "" {
		return ErrEmptyTitle
	}
	return nil
}

func NormalizePriority(priority string) string {
	priority = strings.TrimSpace(strings.ToLower(priority))
	if priority == "" {
		return "normal"
	}
	return priority
}
```

## 5. 添加 service 测试

创建：

```text
internal/service/todo_test.go
```

内容：

```go
package service

import (
	"errors"
	"testing"
)

func TestNormalizeTitle(t *testing.T) {
	tests := []struct {
		name  string
		input string
		want  string
	}{
		{name: "keeps normal title", input: "buy milk", want: "buy milk"},
		{name: "trims spaces", input: "  buy milk  ", want: "buy milk"},
		{name: "keeps empty title empty", input: "", want: ""},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := NormalizeTitle(tt.input)
			if got != tt.want {
				t.Fatalf("NormalizeTitle(%q) = %q, want %q", tt.input, got, tt.want)
			}
		})
	}
}

func TestValidateTitle(t *testing.T) {
	tests := []struct {
		name    string
		title   string
		wantErr error
	}{
		{name: "accepts title", title: "buy milk"},
		{name: "rejects empty title", title: "", wantErr: ErrEmptyTitle},
		{name: "rejects blank title", title: "   ", wantErr: ErrEmptyTitle},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			err := ValidateTitle(tt.title)
			if !errors.Is(err, tt.wantErr) {
				t.Fatalf("ValidateTitle(%q) error = %v, want %v", tt.title, err, tt.wantErr)
			}
		})
	}
}

func TestNormalizePriority(t *testing.T) {
	tests := []struct {
		name  string
		input string
		want  string
	}{
		{name: "uses default", input: "", want: "normal"},
		{name: "trims and lowercases", input: " HIGH ", want: "high"},
		{name: "keeps low", input: "low", want: "low"},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := NormalizePriority(tt.input)
			if got != tt.want {
				t.Fatalf("NormalizePriority(%q) = %q, want %q", tt.input, got, tt.want)
			}
		})
	}
}
```

## 6. 添加 health handler

创建：

```text
internal/handler/health.go
```

内容：

```go
package handler

import (
	"encoding/json"
	"net/http"
)

type HealthResponse struct {
	Status string `json:"status"`
}

func Healthz(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	if err := json.NewEncoder(w).Encode(HealthResponse{Status: "ok"}); err != nil {
		http.Error(w, "encode response", http.StatusInternalServerError)
		return
	}
}
```

## 7. 添加 handler 测试

创建：

```text
internal/handler/health_test.go
```

内容：

```go
package handler

import (
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestHealthz(t *testing.T) {
	req := httptest.NewRequest(http.MethodGet, "/healthz", nil)
	rec := httptest.NewRecorder()

	Healthz(rec, req)

	if rec.Code != http.StatusOK {
		t.Fatalf("status = %d, want %d", rec.Code, http.StatusOK)
	}

	var body HealthResponse
	if err := json.NewDecoder(rec.Body).Decode(&body); err != nil {
		t.Fatalf("decode response: %v", err)
	}

	if body.Status != "ok" {
		t.Fatalf("status body = %q, want %q", body.Status, "ok")
	}
}
```

## 8. 添加版本信息

创建：

```text
internal/version/version.go
```

内容：

```go
package version

var (
	Version = "dev"
	Commit  = "none"
	Date    = "unknown"
)
```

后续构建和部署时可以注入这些变量。

## 9. 添加 .gitignore 和 .env.example

`.gitignore`：

```text
bin/
coverage.out
.env
.env.*
!.env.example
*.pem
*.key
```

`.env.example`：

```text
APP_ENV=local
HTTP_ADDR=:8080
DATABASE_URL=
```

## 10. 添加 golangci-lint 配置

`.golangci.yml`：

```yaml
version: "2"

run:
  timeout: 5m

linters:
  enable:
    - errcheck
    - govet
    - ineffassign
    - staticcheck
    - unused

issues:
  max-issues-per-linter: 0
  max-same-issues: 0
```

## 11. 添加 Makefile

```makefile
.PHONY: fmt fmt-check vet lint test test-race coverage vuln build ci clean

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

## 12. 跑完整门禁

先安装工具：

```bash
go install golang.org/x/vuln/cmd/govulncheck@latest
golangci-lint version
govulncheck -version
```

执行：

```bash
go mod tidy
go fmt ./...
go vet ./...
go test ./...
go test -race ./...
go test -coverprofile=coverage.out ./...
go tool cover -func=coverage.out
go build ./...
golangci-lint run
govulncheck ./...
```

如果有 `make`：

```bash
make ci
```

## 13. 输出项目文档

创建：

```text
docs/quality-gates.md
```

内容模板：

````markdown
# Go 项目质量门禁

## 本地快速检查

```bash
go fmt ./...
go test ./...
go build ./...
```

## PR 前完整检查

```bash
make ci
```

## 检查项

- fmt：检查代码格式。
- vet：Go 官方基础静态检查。
- lint：golangci-lint 静态分析。
- test：单元测试和普通测试。
- race：并发数据竞争检查。
- coverage：生成覆盖率报告。
- vuln：Go 漏洞检查。
- build：确认服务可构建。

## 工具

- Go。
- golangci-lint。
- govulncheck。
````

## 14. 本节验收

你完成后应该能做到：

```bash
go test ./...
go test -race ./...
go build ./...
golangci-lint run
govulncheck ./...
```

如果安装了 make：

```bash
make ci
```

全部通过。

## 15. 本节小结

你现在已经为 `go-cicd-lab` 建立了第一个本地质量门禁系统。

第 3 阶段写 CI 时，你的任务会简单很多：让 CI checkout 代码、安装 Go、安装工具，然后执行 `make ci`。
