# 05：缓存、Artifact、Matrix 与报告

## 1. 本节目标

CI 跑通之后，你要让它更快、更可追溯。

这一节学习：

- Go 依赖缓存。
- 覆盖率报告 artifact。
- 测试 JSON 报告。
- matrix 多版本测试。
- 何时使用并行 job。

## 2. Cache 和 Artifact 的区别

第 0 阶段已经讲过：

```text
cache 是为了跑得快。
artifact 是为了保存本次运行结果。
```

在 Go CI 中：

| 类型 | 示例 | 作用 |
| --- | --- | --- |
| cache | module cache、build cache | 加速后续 CI |
| artifact | coverage.out、test-result.json、bin/server | 保存本次结果 |

cache 不是交付物，不应该依赖 cache 才能构建成功。

## 3. Go 依赖缓存

使用 `actions/setup-go` 时可以启用缓存：

```yaml
- name: Setup Go
  uses: actions/setup-go@v5
  with:
    go-version: "1.25.x"
    cache: true
```

如果你的 `go.sum` 不在根目录，可以指定：

```yaml
- name: Setup Go
  uses: actions/setup-go@v5
  with:
    go-version: "1.25.x"
    cache: true
    cache-dependency-path: subdir/go.sum
```

学习项目通常根目录一个 `go.sum`，不需要额外配置。

## 4. 上传覆盖率 artifact

生成覆盖率：

```bash
go test -coverprofile=coverage.out ./...
```

上传：

```yaml
- name: Upload coverage
  uses: actions/upload-artifact@v4
  with:
    name: coverage
    path: coverage.out
    if-no-files-found: error
```

即使后续不接第三方覆盖率平台，先把 `coverage.out` 保存下来也很有价值。

## 5. 上传测试 JSON

Go 可以输出 JSON 测试结果：

```bash
go test -json ./... > test-results.json
```

workflow：

```yaml
- name: Test JSON
  run: go test -json ./... > test-results.json

- name: Upload test results
  uses: actions/upload-artifact@v4
  with:
    name: test-results
    path: test-results.json
    if-no-files-found: error
```

注意：如果测试失败，重定向文件可能仍然有内容，但 step 会失败。你可以先用基础 `go test ./...`，后面再优化报告。

## 6. always()

如果测试失败，你仍然想上传已有报告，可以用：

```yaml
- name: Upload coverage
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: coverage
    path: coverage.out
    if-no-files-found: ignore
```

`always()` 的意思是前面 step 失败也执行这个 step。

不要滥用。核心检查失败仍然应该让 job 失败。

## 7. Matrix 多版本测试

matrix 可以让同一个 job 在多个变量组合下运行。

例如测试多个 Go 版本：

```yaml
jobs:
  test:
    name: test-go-${{ matrix.go-version }}
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        go-version: ["1.24.x", "1.25.x"]

    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ matrix.go-version }}
          cache: true
      - run: go test ./...
      - run: go build ./...
```

`fail-fast: false` 表示一个版本失败时，不立刻取消其他版本。

这对排查兼容性很有帮助。

## 8. 什么时候需要 matrix

适合：

- 开源库。
- SDK。
- 需要支持多个 Go 版本的组件。
- 跨平台工具。

不一定适合：

- 普通公司内部服务。
- 只在固定 Go 版本构建部署的后端服务。

对 `go-cicd-lab` 来说，matrix 是学习内容，不是第一天必须开启。

## 9. 构建 artifact

如果你构建了二进制：

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o bin/server ./cmd/server
```

可以上传：

```yaml
- name: Upload binary
  uses: actions/upload-artifact@v4
  with:
    name: server-linux-amd64
    path: bin/server
    if-no-files-found: error
```

第 4 阶段会转向 Docker 镜像。第 3 阶段上传二进制主要是为了理解 artifact。

## 10. Step Summary

GitHub Actions 支持把 Markdown 写入 step summary。

示例：

```yaml
- name: Coverage summary
  run: |
    echo "## Coverage" >> "$GITHUB_STEP_SUMMARY"
    go tool cover -func=coverage.out | tail -n 1 >> "$GITHUB_STEP_SUMMARY"
```

这样 workflow 页面会显示简短覆盖率摘要。

## 11. 小练习

完成：

1. 在 CI 中生成 `coverage.out`。
2. 上传 coverage artifact。
3. 使用 `go test -json` 生成测试报告。
4. 上传 test results artifact。
5. 尝试 matrix 测试两个 Go 版本。
6. 观察 CI 耗时变化。

## 12. 本节小结

你现在应该理解：

- cache 用于加速，artifact 用于保存结果。
- `setup-go` 可以缓存 Go 依赖。
- `upload-artifact` 可以保存覆盖率、测试结果和二进制。
- matrix 适合多版本、多系统测试。
- artifact 和 summary 能让 CI 结果更可追溯。

