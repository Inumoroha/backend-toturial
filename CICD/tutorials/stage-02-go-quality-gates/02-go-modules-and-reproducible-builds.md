# 02：Go Modules 与可复现构建

## 1. 为什么先讲依赖

CI 中最常见的失败之一是依赖问题：

```text
本地能编译，CI 下载依赖失败。
同事本地依赖版本不同。
go.sum 没提交。
私有模块没有权限。
```

所以在写测试和 CI 前，先把 Go Modules 理清楚。

## 2. 初始化模块

在项目根目录执行：

```bash
go mod init github.com/your-name/go-cicd-lab
```

这会生成：

```text
go.mod
```

示例：

```go
module github.com/your-name/go-cicd-lab

go 1.25
```

`module` 是你的模块路径。真实项目中通常使用仓库地址。

## 3. go.mod 是什么

`go.mod` 描述模块信息和直接依赖。

示例：

```go
module github.com/your-name/go-cicd-lab

go 1.25

require (
	github.com/gin-gonic/gin v1.10.0
)
```

它回答：

- 当前模块叫什么。
- 当前模块声明的 Go 版本。
- 项目依赖哪些模块。
- 依赖版本是什么。

## 4. go.sum 是什么

`go.sum` 保存模块内容校验和。

它的作用是确认下载到的依赖没有被篡改。

重要规则：

```text
go.mod 和 go.sum 都应该提交到 Git。
```

不要把 `go.sum` 加进 `.gitignore`。

## 5. 添加依赖

例如添加 Gin：

```bash
go get github.com/gin-gonic/gin
```

指定版本：

```bash
go get github.com/gin-gonic/gin@v1.10.0
```

添加依赖后，查看变化：

```bash
git diff go.mod go.sum
```

## 6. 整理依赖

常用命令：

```bash
go mod tidy
```

它会：

- 添加代码实际使用但 `go.mod` 缺少的依赖。
- 移除代码不再使用的依赖。
- 更新 `go.sum`。

建议在提交前执行：

```bash
go mod tidy
git status
```

如果 `go.mod` 或 `go.sum` 发生变化，要确认是否合理。

## 7. 下载依赖

CI 中常用：

```bash
go mod download
```

它会提前下载模块依赖。

这样后面的测试和构建日志更清晰。如果下载依赖失败，问题会更早暴露。

## 8. 校验依赖

可以执行：

```bash
go mod verify
```

它会检查模块缓存中的依赖是否和下载时记录的校验信息匹配。

这不是每个小项目必跑，但理解它对供应链安全有帮助。

## 9. 查看模块图

查看所有依赖：

```bash
go list -m all
```

查看某个依赖为什么被引入：

```bash
go mod why github.com/gin-gonic/gin
```

如果你发现一个奇怪依赖，可以用 `go mod why` 追踪。

## 10. 可复现构建的基本要求

一个适合 CI 的 Go 项目应该满足：

- `go.mod` 已提交。
- `go.sum` 已提交。
- 不依赖开发者本地未提交文件。
- 不依赖本地绝对路径。
- 构建命令清晰。
- 版本信息可以从 commit 或 tag 注入。

最小构建命令：

```bash
go build ./...
```

构建服务二进制：

```bash
go build -o bin/server ./cmd/server
```

Linux 静态二进制常见写法：

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o bin/server ./cmd/server
```

Windows PowerShell 写法：

```powershell
$env:CGO_ENABLED="0"
$env:GOOS="linux"
$env:GOARCH="amd64"
go build -o bin/server ./cmd/server
```

## 11. 注入版本信息

你可以在代码中准备变量：

```go
package version

var (
	Version = "dev"
	Commit  = "none"
	Date    = "unknown"
)
```

构建时注入：

```bash
go build \
  -ldflags "-X 'github.com/your-name/go-cicd-lab/internal/version.Version=v0.1.0' -X 'github.com/your-name/go-cicd-lab/internal/version.Commit=8f3a2c1'" \
  -o bin/server ./cmd/server
```

这样服务可以提供：

```text
GET /version
```

返回当前版本和 commit。后续部署排查会非常有用。

## 12. 小练习

在你的项目中完成：

```bash
go mod init github.com/your-name/go-cicd-lab
go mod tidy
go mod download
go mod verify
go build ./...
```

然后检查：

```bash
git status
git diff go.mod go.sum
```

## 13. 本节小结

你现在应该理解：

- `go.mod` 描述模块和依赖版本。
- `go.sum` 提供依赖校验信息。
- `go mod tidy` 用于整理依赖。
- `go mod download` 常用于 CI 依赖下载。
- 可复现构建要求依赖、命令和版本信息都明确。

