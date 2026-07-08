# 07：并行、Matrix 与 Concurrency

## 1. 本节目标

流水线优化不只是缓存。

有些任务可以并行，有些任务必须串行。

这一节学习：

- 如何拆分 job。
- 如何使用 `needs` 控制依赖。
- 如何使用 matrix。
- 如何取消过时 PR run。
- 如何避免 production 部署并发冲突。

## 2. 串行流水线的问题

低效写法：

```yaml
jobs:
  all:
    runs-on: ubuntu-latest
    steps:
      - run: go test ./...
      - run: golangci-lint run
      - run: govulncheck ./...
      - run: docker build -t app .
      - run: trivy image app
```

问题：

```text
lint、govulncheck、docker build 可能不必全部串行。
一个步骤失败后，其他检查完全没有结果。
开发者需要等待更久才知道全部问题。
```

## 3. 拆分 job

示例：

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true
      - run: go test ./...

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true
      - run: golangci-lint run

  vuln:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true
      - run: go install golang.org/x/vuln/cmd/govulncheck@latest
      - run: govulncheck ./...
```

优点：

- 多个 job 并行。
- 失败结果更清楚。
- 可以设置不同权限。

代价：

- 多次 checkout/setup。
- runner 消耗可能增加。
- 日志分散。

## 4. 使用 needs 控制依赖

镜像构建通常依赖测试通过：

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: go test ./...

  image:
    runs-on: ubuntu-latest
    needs: [test]
    steps:
      - run: docker build -t app .
```

如果有多个前置 job：

```yaml
needs: [test, lint, vuln]
```

建议：

```text
PR 阶段尽快反馈 test/lint。
main 阶段在质量门禁通过后构建镜像。
production 部署必须依赖签名、扫描和审批。
```

## 5. Matrix 策略

测试多个 Go 版本：

```yaml
strategy:
  fail-fast: false
  matrix:
    go-version: ["1.22", "1.23"]

steps:
  - uses: actions/setup-go@v5
    with:
      go-version: ${{ matrix.go-version }}
```

适合 matrix 的场景：

- 多 Go 版本。
- 多操作系统。
- 多数据库版本。
- 多服务模块。

不适合：

- 每个 PR 都跑大量不必要组合。
- 生产部署 job。
- 对外部服务有强依赖的昂贵测试。

## 6. 路径过滤

如果只改文档，不必跑完整镜像构建。

示例：

```yaml
on:
  pull_request:
    paths:
      - "**.go"
      - "go.mod"
      - "go.sum"
      - "Dockerfile"
      - ".github/workflows/**"
```

也可以忽略：

```yaml
on:
  pull_request:
    paths-ignore:
      - "docs/**"
      - "**.md"
```

注意：

```text
路径过滤可能影响 required checks。
如果 GitHub 期望某个检查必须出现，而 workflow 被跳过，PR 可能卡住。
```

## 7. 取消过时的 PR run

当开发者连续 push 多次，旧 run 通常已经没有价值。

使用 `concurrency`：

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

效果：

```text
同一个分支的新 run 会取消旧 run。
节省 runner 时间。
让开发者关注最新结果。
```

## 8. 串行化 production 部署

production 部署不能互相踩踏。

示例：

```yaml
concurrency:
  group: production-deploy
  cancel-in-progress: false
```

含义：

```text
同一时间只允许一个 production deploy。
新的部署等待旧部署结束，而不是取消旧部署。
```

staging 可以更激进：

```yaml
concurrency:
  group: staging-${{ github.ref }}
  cancel-in-progress: true
```

## 9. 并行不是越多越好

过度并行的问题：

- runner 成本增加。
- 外部服务被打爆。
- 日志分散。
- cache 争抢。
- required checks 太多。

判断标准：

```text
是否显著缩短反馈时间？
是否保持排障清晰？
是否控制成本？
是否不会引入部署竞态？
```

## 10. 小练习

改造你的 CI：

1. 将 test、lint、vuln 拆成独立 job。
2. 让 image job `needs: [test, lint, vuln]`。
3. 为 PR workflow 增加 `concurrency cancel-in-progress`。
4. 为 production deploy 增加固定 concurrency group。
5. 比较拆分前后 PR 首次反馈时间。

## 11. 本节小结

你现在应该理解：

- job 拆分能提升并行度和可读性。
- `needs` 用来表达真实依赖。
- matrix 适合验证兼容性，但要控制组合数量。
- PR 可以取消过时 run，production 部署应该串行。
- 并行优化要同时考虑速度、成本和稳定性。

