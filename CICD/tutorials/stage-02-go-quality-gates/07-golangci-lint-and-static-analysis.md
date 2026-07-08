# 07：golangci-lint 与静态分析

## 1. 本节目标

`go vet` 是 Go 官方基础检查，但真实项目通常还会使用更完整的 lint 工具。

这一节学习：

- 安装 `golangci-lint`。
- 运行 `golangci-lint run`。
- 编写 `.golangci.yml`。
- 处理 lint 失败。
- 避免一开始配置过重。

## 2. golangci-lint 是什么

`golangci-lint` 是 Go 项目常用的 lint runner。

它可以并行运行多个 linter，并支持配置文件和缓存。

常用命令：

```bash
golangci-lint run
```

查看版本：

```bash
golangci-lint version
```

## 3. 安装

优先参考官方安装文档：

```text
https://golangci-lint.run/docs/welcome/install/
```

Windows 可以使用 Chocolatey：

```powershell
choco install golangci-lint
```

也可以使用官方提供的二进制安装方式。

安装后检查：

```bash
golangci-lint version
```

## 4. 第一次运行

在项目根目录执行：

```bash
golangci-lint run
```

如果项目还很小，可能没有输出。没有输出通常表示通过。

如果有问题，输出会包含：

```text
文件路径:行号:列号: 问题描述 (linter 名称)
```

不要一看到很多 lint 错误就慌。先从最容易理解的开始修。

## 5. 配置文件

常见配置文件名：

```text
.golangci.yml
.golangci.yaml
.golangci.toml
.golangci.json
```

学习项目推荐使用：

```text
.golangci.yml
```

## 6. 初学推荐配置

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

说明：

- `errcheck`：检查错误是否被忽略。
- `govet`：Go 官方 vet 检查。
- `ineffassign`：检查无效赋值。
- `staticcheck`：常用静态分析。
- `unused`：检查未使用代码。

注意：`golangci-lint` v2 的配置格式和 v1 有差异。新项目建议使用 v2 格式；如果你使用旧版本，请参考对应版本文档。

## 7. 忽略问题要谨慎

有时你确实需要忽略某一行。

示例：

```go
_ = json.NewEncoder(w).Encode(resp) //nolint:errcheck
```

但不要滥用 `//nolint`。

更好的写法通常是处理错误：

```go
if err := json.NewEncoder(w).Encode(resp); err != nil {
	http.Error(w, "encode response", http.StatusInternalServerError)
	return
}
```

规则：

```text
能修就修。
必须忽略时写清楚原因。
```

## 8. 常见 lint 问题

### 未检查错误

```go
json.NewEncoder(w).Encode(resp)
```

修复：

```go
if err := json.NewEncoder(w).Encode(resp); err != nil {
	return
}
```

### 未使用变量

```go
result := doSomething()
```

修复：

- 删除变量。
- 使用变量。
- 如果确实要忽略，使用 `_`，但要谨慎。

### 无效赋值

```go
x := 1
x = 2
return x
```

这里 `x := 1` 可能是无意义赋值。

## 9. lint 和团队风格

lint 不应该变成“折磨开发者”的工具。

好的 lint 配置应该：

- 先覆盖高价值问题。
- 规则数量逐步增加。
- 对历史项目可以分阶段治理。
- 对新代码更严格。

学习项目建议先用基础配置，不要一上来开启所有 linter。

## 10. CI 中如何使用

第 3 阶段会写 CI 配置，现在先知道命令：

```bash
golangci-lint run
```

如果 CI 中没有预装，需要安装或使用官方 Action/镜像。

本阶段目标是本地先跑通。

## 11. 小练习

1. 安装 `golangci-lint`。
2. 在项目根目录创建 `.golangci.yml`。
3. 执行：

```bash
golangci-lint run
```

4. 故意写一段忽略错误的代码，观察 lint 是否提示。
5. 修复 lint 问题。

## 12. 本节小结

你现在应该理解：

- `golangci-lint` 是 Go 项目常用静态分析工具。
- 初学配置要克制，优先开启高价值检查。
- lint 失败应该优先修复，不要随便 `nolint`。
- 本地跑通 `golangci-lint run` 后，CI 才好接入。

