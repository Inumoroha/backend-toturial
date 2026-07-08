# 03：格式化、vet 与构建检查

## 1. 本节目标

这一节学习 Go 项目最基础的三个门禁：

```text
格式化 -> 静态基础检查 -> 构建检查
```

对应命令：

```bash
go fmt ./...
go vet ./...
go build ./...
```

## 2. gofmt 和 go fmt

Go 社区非常重视统一格式。

`gofmt` 是格式化工具，`go fmt` 是对包执行格式化的 Go 命令。

格式化整个项目：

```bash
go fmt ./...
```

它会直接修改文件。

如果你想在 CI 中只检查格式，而不修改文件，可以用：

```bash
gofmt -l .
```

`-l` 会列出格式不符合要求的文件。

更严格一点：

```bash
test -z "$(gofmt -l .)"
```

Windows PowerShell 写法：

```powershell
$files = gofmt -l .
if ($files) {
  $files
  exit 1
}
```

初学阶段，本地用 `go fmt ./...`，CI 中再用格式检查即可。

## 3. go vet

`go vet` 用于报告 Go 代码中可疑的写法。

执行：

```bash
go vet ./...
```

它能发现一些编译器不会报错，但很可能有问题的代码。

例如：

- `fmt.Printf` 参数和格式不匹配。
- 不合理的 struct tag。
- unreachable 或可疑模式。
- copy lock 等风险。

注意：

```text
go vet 不是完整 lint 工具。
它是 Go 官方提供的基础静态检查。
```

更严格的规则后面会交给 `golangci-lint`。

## 4. go build

`go build` 用于确认代码可以编译。

检查所有包：

```bash
go build ./...
```

构建服务入口：

```bash
go build -o bin/server ./cmd/server
```

为什么 `go test` 通过了还要 `go build`？

因为：

- 有些包可能没有测试。
- 服务入口可能没有被测试覆盖。
- 构建目标可能和测试目标不同。
- CI/CD 后续需要真实二进制或镜像。

所以 `go build ./...` 是很基础但重要的门禁。

## 5. 常见构建参数

禁用 CGO：

```bash
CGO_ENABLED=0 go build ./...
```

指定目标系统和架构：

```bash
GOOS=linux GOARCH=amd64 go build -o bin/server ./cmd/server
```

组合使用：

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o bin/server ./cmd/server
```

PowerShell 写法：

```powershell
$env:CGO_ENABLED="0"
$env:GOOS="linux"
$env:GOARCH="amd64"
go build -o bin/server ./cmd/server
```

## 6. 构建输出目录

推荐把本地构建结果放到：

```text
bin/
```

并在 `.gitignore` 中忽略：

```text
bin/
coverage.out
```

不要把本地二进制提交进 Git。

## 7. 最小命令组合

提交前可以先跑：

```bash
go fmt ./...
go vet ./...
go test ./...
go build ./...
```

如果这些都失败，先不要创建 PR。

## 8. 小练习

在 `go-cicd-lab` 里完成：

1. 新增 `.gitignore`：

```text
bin/
coverage.out
```

2. 执行：

```bash
go fmt ./...
go vet ./...
go build ./...
```

3. 故意写一个格式很乱的 Go 文件，再执行：

```bash
go fmt ./...
```

观察文件如何变化。

## 9. 本节小结

你现在应该理解：

- `go fmt ./...` 统一格式。
- `gofmt -l .` 适合在 CI 中检查格式。
- `go vet ./...` 做官方基础静态检查。
- `go build ./...` 确认项目能编译。
- 构建产物不要提交到 Git。

