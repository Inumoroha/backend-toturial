# 01：质量门禁地图

## 1. 什么是质量门禁

质量门禁是进入下一阶段前必须通过的检查。

在第 0 阶段里，我们说过：

```text
PR -> CI -> merge main -> build artifact -> deploy staging -> release production
```

第 2 阶段重点关注 PR 和 main 前半段：

```text
代码是否格式正确？
代码是否能编译？
测试是否通过？
是否存在明显静态问题？
是否存在已知漏洞？
```

如果这些问题在本地和 PR 阶段就能发现，就不要把它们留到部署阶段。

## 2. Go 后端常见质量门禁

| 门禁 | 常用命令 | 解决什么问题 |
| --- | --- | --- |
| 格式化 | `gofmt` / `go fmt ./...` | 统一代码风格 |
| 静态基础检查 | `go vet ./...` | 发现可疑代码和常见错误 |
| 单元测试 | `go test ./...` | 验证函数和模块行为 |
| Race 检查 | `go test -race ./...` | 发现并发数据竞争 |
| 覆盖率 | `go test -coverprofile=coverage.out ./...` | 观察测试覆盖范围 |
| 构建检查 | `go build ./...` | 确认项目能编译 |
| Lint | `golangci-lint run` | 更严格的静态分析 |
| 漏洞检查 | `govulncheck ./...` | 检查 Go 代码和依赖中的可达漏洞 |

这些命令会成为第 3 阶段 CI 的主体。

## 3. 本地命令和 CI 命令要一致

一个常见问题是：

```text
本地不知道该跑什么。
CI 里又写了一套复杂命令。
失败时大家只能反复推 commit 试错。
```

更好的方式：

```text
本地 Makefile 定义统一命令。
CI 直接调用这些命令。
```

例如：

```bash
make ci
```

内部执行：

```text
fmt-check -> vet -> lint -> test -> race -> vuln -> build
```

这样开发者可以在提交前本地跑一遍，减少 PR 上的低级失败。

## 4. 推荐 Go 项目结构

学习项目可以使用这个结构：

```text
go-cicd-lab/
  cmd/
    server/
      main.go
  internal/
    config/
    handler/
    service/
    repository/
  migrations/
  docs/
  scripts/
  go.mod
  go.sum
  Makefile
  .golangci.yml
```

说明：

- `cmd/server`：服务入口。
- `internal`：项目内部包，外部不能直接 import。
- `handler`：HTTP 层。
- `service`：业务逻辑。
- `repository`：数据访问。
- `migrations`：数据库迁移。
- `docs`：项目文档。
- `scripts`：本地辅助脚本，可选。

## 5. 质量门禁和项目分层

不同层适合不同测试：

| 层 | 测试方式 | 说明 |
| --- | --- | --- |
| config | 单元测试 | 验证配置解析、默认值、错误处理 |
| service | 单元测试 | 重点测业务规则 |
| repository | 集成测试 | 需要数据库 |
| handler | HTTP 测试或集成测试 | 验证请求、响应、状态码 |
| cmd/server | 构建和启动测试 | 确认服务能启动 |

初学时优先测试 service 和 handler。

## 6. 质量门禁分层

不是所有检查都必须每次都跑。

建议：

### 提交前本地快速检查

```bash
go fmt ./...
go test ./...
go build ./...
```

### PR 必跑

```bash
go vet ./...
golangci-lint run
go test ./...
go test -race ./...
go build ./...
```

### main 或定时任务跑

```bash
govulncheck ./...
go test -coverprofile=coverage.out ./...
```

### 重点模块或定期跑

```bash
go test -bench=. ./...
go test -fuzz=FuzzName ./...
```

这不是绝对规则。团队可以根据项目规模和耗时调整。

## 7. 质量门禁失败应该怎么处理

遇到失败，不要先怪 CI。

排查顺序：

1. 本地能否复现。
2. 错误发生在哪个命令。
3. 是否依赖环境变量。
4. 是否依赖数据库、Redis 或外部服务。
5. 是否因为测试顺序或并发导致不稳定。
6. 是否因为缓存或依赖版本变化。

好的质量门禁应该让失败原因清楚，而不是只给一个模糊的红叉。

## 8. 小练习

为你的项目写一张表：

| 检查项 | 本地是否跑 | PR 是否跑 | main 是否跑 | 失败是否阻止合并 |
| --- | --- | --- | --- | --- |
| fmt | | | | |
| vet | | | | |
| lint | | | | |
| unit test | | | | |
| integration test | | | | |
| race | | | | |
| coverage | | | | |
| vuln | | | | |
| build | | | | |

## 9. 本节小结

你现在应该理解：

- 质量门禁是 CI/CD 的基础。
- Go 项目常见门禁包括 fmt、vet、test、race、coverage、lint、vuln、build。
- 本地命令和 CI 命令应该尽量一致。
- 第 2 阶段要先把本地质量流水线跑通。

