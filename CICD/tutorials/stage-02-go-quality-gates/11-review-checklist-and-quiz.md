# 11：复习清单与自测题

## 1. 本节目标

用清单和问题确认你是否掌握第 2 阶段。

如果你能本地跑通主要命令，并能解释每个命令的价值，就可以进入第 3 阶段：第一个 CI 流水线。

## 2. 操作清单

### Go Modules

- [ ] 我能解释 `go.mod` 的作用。
- [ ] 我能解释 `go.sum` 的作用。
- [ ] 我知道 `go.mod` 和 `go.sum` 都应该提交到 Git。
- [ ] 我会执行 `go mod tidy`。
- [ ] 我会执行 `go mod download`。
- [ ] 我会执行 `go mod verify`。

### 格式化、vet 和构建

- [ ] 我会执行 `go fmt ./...`。
- [ ] 我知道 `gofmt -l .` 适合做格式检查。
- [ ] 我会执行 `go vet ./...`。
- [ ] 我会执行 `go build ./...`。
- [ ] 我知道本地构建产物不应该提交到 Git。

### 测试

- [ ] 我会编写 `_test.go` 文件。
- [ ] 我会编写 `TestXxx(t *testing.T)`。
- [ ] 我会使用表格驱动测试。
- [ ] 我会执行 `go test ./...`。
- [ ] 我会执行 `go test -run TestName ./path`。
- [ ] 我知道单元测试和集成测试的区别。
- [ ] 我会用 `httptest` 测 HTTP handler。

### 进阶测试

- [ ] 我会执行 `go test -race ./...`。
- [ ] 我会生成 `coverage.out`。
- [ ] 我会用 `go tool cover -func=coverage.out` 查看覆盖率。
- [ ] 我知道 benchmark 解决什么问题。
- [ ] 我知道 fuzzing 适合哪些函数。

### Lint 和安全

- [ ] 我安装了 `golangci-lint`。
- [ ] 我能执行 `golangci-lint run`。
- [ ] 我创建了 `.golangci.yml`。
- [ ] 我安装了 `govulncheck`。
- [ ] 我能执行 `govulncheck ./...`。
- [ ] 我知道 `.env` 不应该提交。
- [ ] 我创建了 `.env.example`。

### 本地质量流水线

- [ ] 我创建了 Makefile。
- [ ] 我能执行 `make test`。
- [ ] 我能执行 `make ci`。
- [ ] 我知道 CI 后续可以复用 Makefile。
- [ ] 我创建了 `docs/quality-gates.md`。

## 3. 自测题

### 题目 1：为什么 CI 前要先统一本地命令？

参考答案：

```text
本地命令统一后，开发者可以在提交前复现 CI 的主要检查。
CI 也可以直接复用 Makefile，减少“本地不知道怎么跑、CI 里另写一套”的问题。
```

### 题目 2：`go test ./...` 和 `go build ./...` 都要跑吗？

参考答案：

```text
建议都跑。
go test 会编译并执行测试相关包，但有些入口或包可能没有被测试覆盖。
go build ./... 可以确认项目整体可构建。
```

### 题目 3：`go vet` 和 `golangci-lint` 有什么区别？

参考答案：

```text
go vet 是 Go 官方基础静态检查，关注可疑代码和常见错误。
golangci-lint 是 lint runner，可以运行多个 linter，覆盖更广的静态分析规则。
```

### 题目 4：为什么表格驱动测试在 Go 中常见？

参考答案：

```text
它适合用统一结构覆盖多组输入和输出。
新增测试 case 很方便，失败时也能通过 t.Run 的名称定位具体场景。
```

### 题目 5：race detector 解决什么问题？

参考答案：

```text
race detector 用于发现并发数据竞争。
当多个 goroutine 同时访问共享变量且至少一个是写操作时，可能导致不可预测行为。
```

### 题目 6：覆盖率越高就代表测试越好吗？

参考答案：

```text
不一定。
覆盖率只能说明代码被执行过，不代表断言有效。
更重要的是核心业务、错误路径、边界输入和关键依赖失败是否被测试覆盖。
```

### 题目 7：`govulncheck` 和普通依赖扫描有什么不同？

参考答案：

```text
govulncheck 会结合 Go 代码和依赖，分析漏洞是否可能被代码调用到。
它不只是列出依赖中所有漏洞，还关注可达性。
```

### 题目 8：为什么 `.env.example` 可以提交，而 `.env` 不应该提交？

参考答案：

```text
.env.example 只提供配置字段模板，不包含真实密钥。
.env 往往包含数据库密码、token、API key 等敏感信息，不能进入 Git。
```

## 4. 情景题

### 情景 1

PR 中 `go test ./...` 通过，但 `go test -race ./...` 失败。

你会怎么理解？

参考思路：

```text
普通测试通过只能说明功能结果在该次运行中正确。
race 失败说明并发访问共享数据存在数据竞争，未来可能出现随机错误。
应该定位 race 报告中的读写位置，用 mutex、channel、atomic 或减少共享状态来修复。
```

### 情景 2

`golangci-lint run` 报了很多历史问题，团队不想一次性修完。

你会怎么处理？

参考思路：

```text
不要一上来开启过多规则。
先保留高价值 linter，逐步治理。
也可以先限制新代码，或按包分批修复。
不要为了通过 CI 大量添加无解释的 nolint。
```

### 情景 3

`govulncheck ./...` 报出依赖漏洞，但升级依赖会破坏兼容。

你会怎么处理？

参考思路：

```text
先判断漏洞是否可达和影响范围。
查看修复版本和变更说明。
如果不能立即升级，记录风险、添加临时缓解措施，并安排兼容性改造。
高危且可达漏洞应该阻断发布。
```

## 5. 第 2 阶段最终作业

在 `go-cicd-lab` 中完成：

```text
go.mod
go.sum
.gitignore
.env.example
.golangci.yml
Makefile
docs/quality-gates.md
internal/service/*_test.go
internal/handler/*_test.go
```

并保证这些命令通过：

```bash
go mod tidy
go fmt ./...
go vet ./...
go test ./...
go test -race ./...
go test -coverprofile=coverage.out ./...
go build ./...
golangci-lint run
govulncheck ./...
```

有 `make` 时：

```bash
make ci
```

## 6. 进入下一阶段的标准

当你能做到下面几件事，就可以进入第 3 阶段：

1. 能解释每个质量门禁命令的作用。
2. 能为 service 和 handler 编写基础测试。
3. 能本地跑通完整质量检查。
4. 能用 Makefile 统一本地检查入口。
5. 能说明哪些检查适合 PR，哪些适合 main 或定时任务。

第 3 阶段会把这些本地命令搬进 GitHub Actions 或 GitLab CI，形成你的第一条真实 CI 流水线。

