# 阶段 2：Go 项目的质量门禁

> 本阶段目标：把一个 Go 后端项目整理成“适合自动化检查”的状态。等你进入第 3 阶段写 CI 配置时，CI 只是在执行本阶段已经验证过的本地命令。

## 学习顺序

请按下面顺序学习：

1. [01-quality-gate-map.md](./01-quality-gate-map.md)
   - 理解什么是质量门禁。
   - 建立 Go 后端项目从本地检查到 CI 检查的整体地图。

2. [02-go-modules-and-reproducible-builds.md](./02-go-modules-and-reproducible-builds.md)
   - 学习 Go Modules、`go.mod`、`go.sum`。
   - 理解依赖可复现和构建可复现。

3. [03-format-vet-and-build.md](./03-format-vet-and-build.md)
   - 学习 `gofmt`、`go fmt`、`go vet`、`go build`。
   - 建立最基础的代码检查命令。

4. [04-unit-tests-and-table-driven-tests.md](./04-unit-tests-and-table-driven-tests.md)
   - 学习 Go 单元测试。
   - 掌握表格驱动测试，这是 Go 项目里非常常见的测试写法。

5. [05-http-and-integration-tests.md](./05-http-and-integration-tests.md)
   - 学习 HTTP handler 测试。
   - 理解集成测试和测试依赖管理。

6. [06-race-coverage-benchmark-and-fuzz.md](./06-race-coverage-benchmark-and-fuzz.md)
   - 学习 race detector、覆盖率、benchmark 和 fuzzing。
   - 理解哪些检查适合每次 PR 跑，哪些适合定期或重点场景跑。

7. [07-golangci-lint-and-static-analysis.md](./07-golangci-lint-and-static-analysis.md)
   - 学习 `golangci-lint`。
   - 编写一个适合初学项目的 lint 配置。

8. [08-security-and-dependency-checks.md](./08-security-and-dependency-checks.md)
   - 学习 `govulncheck`、依赖升级和 secret 防护。
   - 建立 Go 项目的基础安全门禁。

9. [09-makefile-and-local-quality-pipeline.md](./09-makefile-and-local-quality-pipeline.md)
   - 用 Makefile 把所有本地命令串起来。
   - 为第 3 阶段 CI 流水线准备统一入口。

10. [10-practice-build-quality-gates-for-go-cicd-lab.md](./10-practice-build-quality-gates-for-go-cicd-lab.md)
    - 为 `go-cicd-lab` 落地完整质量门禁。
    - 输出可被 CI 复用的命令和文档。

11. [11-review-checklist-and-quiz.md](./11-review-checklist-and-quiz.md)
    - 用清单和自测题检查是否可以进入第 3 阶段。

## 本阶段建议时间

- 快速学习：3 到 5 天。
- 扎实学习：2 周。
- 推荐方式：边学边把命令写进你的 `go-cicd-lab` 项目。

## 本阶段你要准备什么

- 已安装 Go。
- 已完成第 1 阶段的 Git/PR 基础。
- 准备一个 Go 练习仓库，推荐继续使用 `go-cicd-lab`。

检查 Go 是否可用：

```bash
go version
go env GOPATH
go env GOMODCACHE
```

## 学完后你应该能做到

- 用 `go test ./...` 跑完整项目测试。
- 用 `go test -race ./...` 检查并发数据竞争。
- 用覆盖率报告判断测试缺口。
- 用 `go vet ./...` 和 `golangci-lint run` 做静态检查。
- 用 `govulncheck ./...` 做 Go 依赖和代码可达漏洞检查。
- 用 `go build ./...` 保证项目可以构建。
- 用 Makefile 提供一条本地质量门禁命令。

## 推荐官方资料

- Go command：<https://pkg.go.dev/cmd/go>
- Go testing：<https://pkg.go.dev/testing>
- Go race detector：<https://go.dev/doc/articles/race_detector>
- Go fuzzing：<https://go.dev/doc/tutorial/fuzz>
- Go vulnerability management：<https://go.dev/doc/security/vuln/>
- golangci-lint：<https://golangci-lint.run/>

