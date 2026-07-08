# 08：复习清单与自测题

## 1. 本节目标

用清单和问题检查你是否掌握第 1 阶段。

如果大部分问题都能用自己的话回答，就可以进入第 2 阶段：Go 项目的质量门禁。

## 2. 操作清单

### Git 基础

- [ ] 我能解释工作区、暂存区、本地仓库、远程仓库。
- [ ] 我能使用 `git status` 查看当前状态。
- [ ] 我能使用 `git add` 选择要提交的文件。
- [ ] 我能使用 `git commit` 创建提交。
- [ ] 我能使用 `git log --oneline` 查看历史。
- [ ] 我知道 commit hash 在 CI/CD 中的作用。

### 分支与同步

- [ ] 我能从 main 创建 feature 分支。
- [ ] 我能推送新分支到远程。
- [ ] 我能更新本地 main。
- [ ] 我知道 merge 和 rebase 的区别。
- [ ] 我知道不要 rebase 公共分支。
- [ ] 我能解决简单冲突。

### PR/MR

- [ ] 我能创建 Pull Request 或 Merge Request。
- [ ] 我能写清楚 PR 描述。
- [ ] 我知道 CI 和 code review 的分工。
- [ ] 我知道 required checks 的作用。
- [ ] 我知道 Squash Merge 的适用场景。

### 分支策略与发布

- [ ] 我知道 Trunk-Based Development 是什么。
- [ ] 我知道 Git Flow 为什么更重。
- [ ] 我能设计 main、feat/*、fix/* 的规则。
- [ ] 我能创建 annotated tag。
- [ ] 我知道语义化版本的 MAJOR、MINOR、PATCH。
- [ ] 我知道 tag 可以触发 release pipeline。

### 仓库规则与安全

- [ ] 我知道 main 分支应该受保护。
- [ ] 我知道生产 secret 不应该暴露给 PR。
- [ ] 我知道 fork PR 有安全风险。
- [ ] 我能为 PR、push main、tag 设计不同流水线动作。
- [ ] 我知道最小权限原则。

## 3. 自测题

### 题目 1：为什么不要直接在 main 上开发？

参考答案：

```text
main 通常代表主干和稳定状态，很多 CI/CD 流程会基于 main 构建镜像或部署 staging。
直接在 main 上开发会绕过 PR、CI、review 和分支保护，增加破坏主干的风险。
```

### 题目 2：commit hash 在 CI/CD 中有什么作用？

参考答案：

```text
commit hash 可以精确定位一次代码状态。
它常用于标记镜像、追踪构建、审计发布和回滚版本。
```

### 题目 3：merge 和 rebase 的区别是什么？

参考答案：

```text
merge 会保留分支历史，并产生合并提交，不改写已有 commit。
rebase 会把当前分支提交搬到目标分支之后，会改写 commit hash，使历史更线性。
```

### 题目 4：PR 阶段适合触发哪些检查？

参考答案：

```text
适合触发格式检查、lint、单元测试、集成测试、构建检查和安全检查。
不适合直接部署生产，也不应该暴露生产 secret。
```

### 题目 5：Trunk-Based Development 的核心是什么？

参考答案：

```text
main 是主干。
开发者从 main 创建短生命周期分支，小步提交，通过 PR 和 CI 快速合并回 main。
```

### 题目 6：为什么生产发布要使用 tag？

参考答案：

```text
tag 可以把一次正式发布绑定到明确 commit。
这样生产版本可追溯、可审计、可回滚，也可以触发 release pipeline。
```

### 题目 7：为什么 PR from fork 不能随便拿到 secret？

参考答案：

```text
外部 PR 中的代码可能是恶意的，可能尝试读取或打印流水线 secret。
所以生产 secret 应该只在受信任分支、tag 或审批后的部署 job 中使用。
```

### 题目 8：push main 和 push tag 分别适合做什么？

参考答案：

```text
push main 适合运行完整 CI、构建镜像、部署 staging。
push tag 适合创建 release、发布正式版本、审批后部署 production。
```

## 4. 情景题

### 情景 1

同事直接 push 到 main，导致 staging 自动部署失败。

你会怎么改进流程？

参考思路：

```text
启用 main 分支保护。
禁止直接 push。
要求所有变更走 PR。
配置 required checks。
至少 1 个 reviewer approval。
main 合并后再部署 staging。
```

### 情景 2

一个 PR 改了业务逻辑、数据库迁移、CI 配置和大量格式化文件，review 很困难。

你会建议怎么处理？

参考思路：

```text
拆分 PR。
把纯格式化、CI 配置、数据库迁移、业务逻辑分开提交或分成多个 PR。
每个 PR 只解决一个清晰问题，降低 review 和回滚成本。
```

### 情景 3

生产现在运行 `v0.3.0`，main 已经有很多新功能。线上有紧急 bug 需要修复，但你不想把 main 的所有新功能一起发到生产。

你会怎么办？

参考思路：

```text
从 `v0.3.0` 对应 commit 创建 hotfix 分支，修复 bug 后打 `v0.3.1`。
同时评估是否需要把修复合回 main，避免后续版本丢失修复。
```

## 5. 第 1 阶段最终作业

请完成一份文档：

```text
docs/git-workflow.md
```

要求包含：

- 分支策略。
- 分支命名。
- PR 规则。
- main 保护规则。
- commit message 规则。
- tag 发布规则。
- hotfix 流程。
- CI/CD 触发规则。
- secret 使用边界。

## 6. 进入下一阶段的标准

当你能做到下面几件事，就可以进入第 2 阶段：

1. 独立完成一次 feature 分支开发和 PR。
2. 能解释 main 为什么必须保护。
3. 能设计 PR、push main、tag 对应的 CI/CD 触发动作。
4. 能用 tag 标记一次发布。
5. 能写出项目的 Git 协作规则。

第 2 阶段会进入 Go 项目质量门禁：测试、lint、race、coverage、构建和安全扫描。那时你会把本阶段的 PR 流程接上真正的自动化检查。

