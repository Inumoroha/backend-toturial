# 10：复习清单与自测题

## 1. 本节目标

用清单和问题确认你是否掌握第 3 阶段。

如果你能独立给 Go 项目接入 PR CI，就可以进入第 4 阶段：Docker、镜像与制品管理。

## 2. 操作清单

### Workflow 基础

- [ ] 我知道 GitHub Actions workflow 文件放在 `.github/workflows/`。
- [ ] 我能写出最小 `ci.yml`。
- [ ] 我知道 `on` 用来定义触发器。
- [ ] 我知道 job 和 step 的区别。
- [ ] 我知道 `runs-on` 表示 runner 环境。
- [ ] 我知道 step 失败会让 job 失败。

### Go CI

- [ ] 我能在 CI 中使用 `actions/checkout`。
- [ ] 我能在 CI 中使用 `actions/setup-go`。
- [ ] 我能执行 `go mod download`。
- [ ] 我能执行 `go vet ./...`。
- [ ] 我能执行 `go test ./...`。
- [ ] 我能执行 `go test -race ./...`。
- [ ] 我能执行 `go build ./...`。
- [ ] 我能让 CI 复用 Makefile。

### Lint、安全和报告

- [ ] 我能接入 `golangci-lint`。
- [ ] 我能接入 `govulncheck`。
- [ ] 我能生成 `coverage.out`。
- [ ] 我能上传 artifact。
- [ ] 我知道 cache 和 artifact 的区别。
- [ ] 我知道 matrix 适合什么场景。

### 权限和分支保护

- [ ] 我知道 `permissions: contents: read` 的意义。
- [ ] 我知道 PR CI 不应该使用生产 secret。
- [ ] 我知道为什么要谨慎使用 `pull_request_target`。
- [ ] 我能配置 required status checks。
- [ ] 我知道 job name 改变可能影响 required checks。

### 排障

- [ ] 我会从 workflow run 找到失败 job。
- [ ] 我会从 job 找到失败 step。
- [ ] 我会根据日志定位失败命令。
- [ ] 我会比较本地和 CI 的 Go 版本。
- [ ] 我会处理 `go.sum` 缺失问题。

## 3. 自测题

### 题目 1：PR CI 主要解决什么问题？

参考答案：

```text
PR CI 主要判断一次代码变更是否具备合并资格。
它通过自动执行测试、静态检查、构建和安全扫描，把明显问题挡在 main 分支之前。
```

### 题目 2：为什么每个 job 通常都要 checkout？

参考答案：

```text
不同 job 默认运行在独立 runner 环境中。
runner 一开始没有项目源码，所以每个需要访问代码的 job 都要 checkout。
```

### 题目 3：cache 和 artifact 有什么区别？

参考答案：

```text
cache 用于加速后续运行，例如 Go module cache。
artifact 用于保存本次运行结果，例如 coverage.out、测试报告、二进制。
```

### 题目 4：为什么建议显式写 `permissions: contents: read`？

参考答案：

```text
这是最小权限原则。
普通 CI 只需要读取代码，不需要写仓库、发包或部署。
显式限制权限可以降低 workflow 或第三方 action 被滥用时的影响。
```

### 题目 5：什么时候使用 matrix？

参考答案：

```text
当你需要在多个 Go 版本、多个操作系统或多个配置下运行同一组检查时使用 matrix。
普通内部后端服务可以先不使用，等有兼容性需求再加入。
```

### 题目 6：为什么不建议 PR CI 使用生产 secret？

参考答案：

```text
PR 中的代码会在 CI 中执行。
如果 PR 来自不受信任来源，恶意代码可能尝试读取或打印 secret。
生产 secret 应该只给受信任分支、tag、部署 job 或受保护环境使用。
```

### 题目 7：CI 本地不复现时你会检查什么？

参考答案：

```text
检查 Go 版本、操作系统、环境变量、依赖文件、是否有未提交文件、是否依赖本地服务、缓存状态和执行命令是否一致。
```

### 题目 8：为什么 required check 名称要稳定？

参考答案：

```text
分支保护规则会绑定具体 check 名称。
如果 workflow name 或 job name 改变，原来的 required check 可能不再匹配，导致 PR 一直等待或保护失效。
```

## 4. 情景题

### 情景 1

PR 页面显示 CI 没有运行。

你会检查什么？

参考思路：

```text
检查 workflow 文件路径是否是 .github/workflows/*.yml。
检查 YAML 是否提交。
检查 on.pull_request.branches 是否匹配目标分支。
检查 PR 目标是不是 main。
检查 Actions 是否被仓库禁用。
```

### 情景 2

CI 中 `go test ./...` 失败，但你本地通过。

你会怎么排查？

参考思路：

```text
先看失败测试和日志。
比较本地和 CI 的 Go 版本。
确认 go.mod/go.sum 是否提交。
检查测试是否依赖环境变量、当前时间、测试顺序、本地数据库或外网。
用 go test -count=1 ./... 尝试复现。
```

### 情景 3

团队希望 CI 更快，有人建议删除 race test。

你会如何回应？

参考思路：

```text
先看耗时数据。
如果 race test 确实很慢，可以考虑只对核心包、main 分支或定时任务运行。
不应该在没有替代方案的情况下直接删除关键并发检查。
```

## 5. 第 3 阶段最终作业

在 `go-cicd-lab` 中完成：

```text
.github/workflows/ci.yml
docs/ci.md
```

CI 要求：

- PR 到 main 自动运行。
- push main 自动运行。
- workflow_dispatch 可手动运行。
- 至少包含 test、lint、vuln 三类检查。
- 上传 coverage artifact。
- 设置最小权限。
- main 分支启用 required status checks。

## 6. 进入下一阶段的标准

当你能做到下面几件事，就可以进入第 4 阶段：

1. 独立写出 Go 项目的 GitHub Actions CI。
2. 能解释 workflow、job、step、runner。
3. 能让 PR CI 阻止坏代码进入 main。
4. 能上传覆盖率 artifact。
5. 能排查常见 CI 失败。
6. 能说明 PR CI 的权限和 secret 边界。

第 4 阶段会学习 Docker、镜像、镜像标签和制品仓库。那时 CI 会从“检查代码”进入“构建可发布制品”。

