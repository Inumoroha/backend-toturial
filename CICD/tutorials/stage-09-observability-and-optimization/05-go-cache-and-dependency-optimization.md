# 05：Go 缓存与依赖优化

## 1. 本节目标

Go 项目 CI 中常见耗时：

```text
下载模块
编译 package
运行测试
安装工具
```

这一节学习如何用缓存降低重复工作，同时避免缓存带来安全和稳定性问题。

## 2. Go 的两个重要缓存

Go 常见缓存：

```text
GOMODCACHE：模块下载缓存。
GOCACHE：构建缓存。
```

查看路径：

```bash
go env GOMODCACHE
go env GOCACHE
```

本地清理缓存：

```bash
go clean -modcache
go clean -cache
```

CI 中通常不需要手动清理，除非排查缓存问题。

## 3. 使用 setup-go 内置缓存

推荐起点：

```yaml
- uses: actions/setup-go@v5
  with:
    go-version-file: go.mod
    cache: true
```

如果项目有多个 `go.sum`：

```yaml
- uses: actions/setup-go@v5
  with:
    go-version-file: go.mod
    cache: true
    cache-dependency-path: |
      go.sum
      tools/go.sum
```

然后：

```yaml
- name: Download modules
  run: go mod download
```

## 4. 什么时候使用 actions/cache

如果内置缓存无法满足需求，可以手动配置：

```yaml
- name: Go cache
  uses: actions/cache@v4
  with:
    path: |
      ~/.cache/go-build
      ~/go/pkg/mod
    key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
    restore-keys: |
      ${{ runner.os }}-go-
```

含义：

- `key` 完全匹配时命中最好。
- `restore-keys` 可使用旧缓存作为降级。
- `hashFiles('**/go.sum')` 让依赖变化时生成新缓存。

注意：

```text
不要同时无脑使用 setup-go cache 和 actions/cache 缓存同一路径。
先用简单方案，确实不够再手动控制。
```

## 5. 缓存命中率怎么观察

GitHub Actions 日志会显示 cache hit/miss。

你也可以手动输出：

```yaml
- name: Go cache
  id: go-cache
  uses: actions/cache@v4
  with:
    path: |
      ~/.cache/go-build
      ~/go/pkg/mod
    key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}

- name: Cache summary
  run: |
    echo "Go cache hit: ${{ steps.go-cache.outputs.cache-hit }}" >> "$GITHUB_STEP_SUMMARY"
```

记录：

```text
cache hit
cache miss
partial restore
download duration
test duration
```

## 6. 工具安装优化

常见写法：

```yaml
- name: Install tools
  run: |
    go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
    go install golang.org/x/vuln/cmd/govulncheck@latest
```

问题：

- `latest` 不可复现。
- 每次安装可能慢。
- 新版本可能突然破坏 CI。

更好：

```yaml
- name: Install govulncheck
  run: go install golang.org/x/vuln/cmd/govulncheck@v1.1.4
```

版本号请按项目需要和官方发布确认。

也可以使用专门的 action，例如 `golangci-lint-action`，但仍要注意版本固定。

## 7. 私有依赖与缓存风险

如果 CI 下载私有 Go module，要注意：

- 缓存中不应包含 token。
- 不要把 `.netrc`、Git 凭证写入缓存路径。
- fork PR 不应访问私有依赖 secret。
- 只给下载依赖的 job 最小权限。

缓存不是 secret 存储。

如果怀疑缓存污染：

```text
删除相关 cache。
轮换凭证。
重新跑干净构建。
```

## 8. PR 与 main 的缓存策略

建议：

```text
PR：尽量复用 main 缓存，快速反馈。
main：负责产生稳定缓存。
release：优先可复现，不依赖不可信缓存。
```

对于不可信 fork PR：

```text
不要让它写入高信任缓存。
不要让它读取敏感凭证。
不要因为缓存命中而跳过安全检查。
```

## 9. 优化验证

改动前记录：

```text
go mod download 平均耗时
go test 平均耗时
CI 总耗时 p50/p95
cache hit rate
```

改动后记录：

```text
连续 20 次运行中命中多少次？
是否出现缓存导致的奇怪失败？
总耗时是否明显下降？
```

## 10. 小练习

完成：

1. 查看本地 `go env GOMODCACHE` 和 `go env GOCACHE`。
2. 在 `ci.yml` 中启用 `setup-go cache: true`。
3. 在 job summary 中输出缓存命中情况。
4. 比较启用前后 10 次 CI 的耗时。
5. 写出你的 cache key 策略。

## 11. 本节小结

你现在应该理解：

- Go module cache 和 build cache 能显著减少重复工作。
- `setup-go cache: true` 是最简单的起点。
- 手动 `actions/cache` 要明确 key、restore key 和路径。
- 工具版本要固定，避免 `latest` 带来不可复现。
- 缓存不能包含 secret，也不能替代安全检查。

