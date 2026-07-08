# 阶段 1：Git、分支策略与协作基础

> 本阶段目标：掌握 CI/CD 的入口，也就是代码如何进入仓库、如何被评审、如何合并、如何通过 tag 和分支事件触发后续流水线。

## 学习顺序

请按下面顺序学习：

1. [01-git-core-mental-model.md](./01-git-core-mental-model.md)
   - 建立 Git 的基本心智模型。
   - 理解工作区、暂存区、本地仓库、远程仓库。

2. [02-daily-git-workflow.md](./02-daily-git-workflow.md)
   - 学习每天都会用到的 Git 命令。
   - 完成从创建分支到提交、推送的完整流程。

3. [03-branching-strategies.md](./03-branching-strategies.md)
   - 理解主干开发、Git Flow、环境分支。
   - 学会为 Go 后端项目选择简单可靠的分支策略。

4. [04-pull-request-and-code-review.md](./04-pull-request-and-code-review.md)
   - 理解 Pull Request/Merge Request。
   - 学习 PR 描述、代码评审、合并策略和状态检查。

5. [05-tags-releases-and-versioning.md](./05-tags-releases-and-versioning.md)
   - 学习 tag、release、语义化版本。
   - 理解为什么生产发布应该绑定明确版本。

6. [06-repository-rules-and-cicd-triggers.md](./06-repository-rules-and-cicd-triggers.md)
   - 学习分支保护、CODEOWNERS、权限规则。
   - 理解 Git 事件如何触发 CI/CD。

7. [07-practice-design-your-git-workflow.md](./07-practice-design-your-git-workflow.md)
   - 为 `go-cicd-lab` 设计一套实际可用的 Git 协作流程。
   - 输出分支规则、PR 规则、发布规则和回滚规则。

8. [08-review-checklist-and-quiz.md](./08-review-checklist-and-quiz.md)
   - 通过清单和自测题确认是否可以进入第 2 阶段。

## 本阶段建议时间

- 快速学习：1 到 2 天。
- 扎实学习：1 周。
- 推荐方式：每天学习 1 篇，并亲手执行里面的命令。

## 本阶段你要准备什么

- 安装 Git。
- 准备一个 GitHub 或 GitLab 账号。
- 准备一个练习仓库，推荐命名为 `go-cicd-lab`。

如果你暂时没有远程仓库，也可以先在本地练习 Git 命令。但本阶段后半部分涉及 PR、分支保护、tag 发布，最好配合 GitHub 或 GitLab 操作。

## 学完后你应该能做到

- 熟练完成 clone、branch、commit、push、pull、merge、rebase、tag。
- 知道什么时候用 merge，什么时候用 rebase。
- 能解释 trunk-based development 和 Git Flow 的区别。
- 能为 Go 后端项目设计 PR 流程。
- 能用 tag 表示一次正式发布。
- 能说明 PR、push main、tag 分别适合触发哪些流水线。

