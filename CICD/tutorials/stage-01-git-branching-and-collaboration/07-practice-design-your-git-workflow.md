# 07：实战，设计你的 Git 协作流程

## 1. 本节目标

这一节把第 1 阶段的知识整合成一个可执行方案。

你要为 `go-cicd-lab` 输出一份 Git 协作流程文档：

```text
分支怎么建？
PR 怎么提？
main 怎么保护？
tag 怎么发布？
hotfix 怎么处理？
CI/CD 怎么触发？
```

## 2. 推荐最终方案

学习项目推荐：

```text
分支策略：Trunk-Based Development
主干分支：main
功能分支：feat/*
修复分支：fix/*
CI/CD 分支：ci/*
文档分支：docs/*
正式发布：tag v*
合并方式：Squash Merge
```

## 3. 初始化练习仓库

如果还没有仓库，可以先创建：

```bash
mkdir go-cicd-lab
cd go-cicd-lab
git init
echo "# go-cicd-lab" > README.md
git add README.md
git commit -m "docs: add project readme"
```

如果你已经有远程仓库：

```bash
git remote add origin https://github.com/your-name/go-cicd-lab.git
git branch -M main
git push -u origin main
```

如果远程已经存在，用 clone：

```bash
git clone https://github.com/your-name/go-cicd-lab.git
cd go-cicd-lab
```

## 4. 创建一次 feature 流程

```bash
git switch main
git pull origin main
git switch -c feat/project-structure
mkdir -p cmd/server internal/handler internal/service internal/repository
echo "package main" > cmd/server/main.go
git status
git add .
git commit -m "feat: add initial project structure"
git push -u origin feat/project-structure
```

Windows PowerShell 中如果没有 `mkdir -p`，可以用：

```powershell
New-Item -ItemType Directory -Force cmd/server,internal/handler,internal/service,internal/repository
Set-Content -Path cmd/server/main.go -Value "package main"
```

然后到 GitHub/GitLab 创建 PR。

## 5. 写一份 PR 描述

```markdown
## 变更内容

- 添加 Go 项目初始目录结构。
- 添加 `cmd/server/main.go` 作为服务入口占位。

## 为什么需要这个变更

- 为后续 Go API、测试和 CI/CD 流水线提供基础结构。

## 如何验证

- [ ] 检查目录结构是否符合预期。
- [ ] 后续接入 Go module 后运行 `go test ./...`。

## 风险点

- 当前只是目录初始化，没有运行时逻辑。

## 回滚方式

- revert 本 PR。
```

## 6. 设计 main 保护规则

在远程平台中配置：

```text
main:
  - Require a pull request before merging
  - Require approvals: 1
  - Require status checks to pass before merging
  - Do not allow force pushes
  - Do not allow deletion
```

如果你是个人练习仓库，没有可用 reviewer，也可以先把 approval 规则记在文档里，等有团队时再启用。

## 7. 设计 commit message 规则

推荐：

```text
feat: 新功能
fix: bug 修复
docs: 文档
test: 测试
ci: CI/CD 配置
chore: 杂项维护
refactor: 重构
```

示例：

```text
feat: add todo creation api
fix: handle invalid todo id
test: add todo service tests
ci: add pull request checks
docs: add local setup guide
```

## 8. 设计 tag 发布规则

发布流程：

```bash
git switch main
git pull origin main
git tag -a v0.1.0 -m "release v0.1.0"
git push origin v0.1.0
```

规则：

```text
只有 main 上的 commit 可以打正式 tag。
tag 格式必须是 vMAJOR.MINOR.PATCH。
tag 触发 release pipeline。
production 部署必须审批。
发布后不移动 tag。
```

## 9. 设计 hotfix 流程

生产紧急修复流程：

```text
确认当前生产版本
-> 从 main 或生产 tag 创建 fix 分支
-> 修复问题
-> PR + CI
-> 合并 main
-> 打 patch tag
-> 发布 production
-> 验证
```

示例：

```bash
git switch main
git pull origin main
git switch -c fix/login-timeout

# 修改代码

git add .
git commit -m "fix: handle login timeout"
git push -u origin fix/login-timeout
```

发布：

```bash
git switch main
git pull origin main
git tag -a v0.1.1 -m "release v0.1.1"
git push origin v0.1.1
```

## 10. 输出最终文档

请在你的项目中创建：

```text
docs/git-workflow.md
```

内容可以参考：

```markdown
# Git 协作流程

## 分支策略

- 使用 Trunk-Based Development。
- `main` 是唯一长期主干分支。
- 所有变更通过短生命周期分支提交。

## 分支命名

- `feat/*`：新功能。
- `fix/*`：bug 修复。
- `docs/*`：文档。
- `test/*`：测试。
- `ci/*`：CI/CD。
- `chore/*`：维护任务。

## PR 规则

- 所有变更必须通过 PR。
- PR 必须描述变更内容、验证方式、风险点和回滚方式。
- PR 必须通过 required checks。
- 合并方式使用 Squash Merge。

## main 规则

- 禁止直接 push。
- 禁止 force push。
- 禁止删除。
- main 必须保持可构建、可测试、可部署。

## 发布规则

- 正式发布使用 annotated tag。
- tag 格式：`vMAJOR.MINOR.PATCH`。
- 只有 main 上的 commit 可以打正式 tag。
- tag 触发 release pipeline。

## CI/CD 触发

- PR：lint、test、build。
- push main：build image、deploy staging。
- tag v*：release、approval、deploy production。
- manual：rollback 或 redeploy。

## Hotfix

- 从 main 或当前生产 tag 创建 `fix/*` 分支。
- 修复后通过 PR 合并 main。
- 发布 patch 版本。
- 发布后验证并记录。
```

## 11. 本节验收

完成后你应该拥有：

- 一个练习仓库。
- 一个 feature 分支。
- 一个 PR。
- 一份 `docs/git-workflow.md`。
- 一套 main 分支保护规则。
- 一套 tag 发布规则。

## 12. 本节小结

你现在已经把 Git 基础能力转换成了 CI/CD 可用的协作入口。

下一阶段学习 Go 项目的质量门禁时，PR 触发 CI、main 触发构建、tag 触发发布就会变得非常自然。

