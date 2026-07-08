# 02：日常 Git 工作流

## 1. 本节目标

这一节学习一个后端工程师每天都会重复的 Git 流程：

```text
更新 main
-> 创建功能分支
-> 修改代码
-> 提交 commit
-> 推送分支
-> 创建 PR
-> 根据反馈修改
-> 合并
```

先把这条路线练熟，后面 CI/CD 才有稳定入口。

## 2. 克隆仓库

如果远程仓库已经存在：

```bash
git clone https://github.com/your-name/go-cicd-lab.git
cd go-cicd-lab
```

查看远程地址：

```bash
git remote -v
```

## 3. 更新 main

开始新任务前，先保证本地 `main` 是最新的：

```bash
git switch main
git pull origin main
```

如果你习惯更明确地分两步：

```bash
git fetch origin
git merge origin/main
```

## 4. 创建功能分支

不要直接在 main 上开发。

创建分支：

```bash
git switch -c feat/todo-priority
```

推荐分支命名：

| 类型 | 示例 | 说明 |
| --- | --- | --- |
| feat | `feat/todo-priority` | 新功能 |
| fix | `fix/login-timeout` | 修复 bug |
| docs | `docs/update-readme` | 文档 |
| refactor | `refactor/user-service` | 重构 |
| chore | `chore/update-deps` | 杂项维护 |
| ci | `ci/add-github-actions` | CI/CD 配置 |

命名原则：

- 小写。
- 使用短横线。
- 能看出目的。
- 不要太长。

## 5. 修改代码并查看差异

查看状态：

```bash
git status
```

查看具体改了什么：

```bash
git diff
```

查看暂存区里的改动：

```bash
git diff --cached
```

## 6. 提交修改

把文件加入暂存区：

```bash
git add internal/handler/todo.go
git add internal/service/todo.go
```

提交：

```bash
git commit -m "feat: add todo priority"
```

推荐使用 Conventional Commits 风格：

```text
feat: add todo priority
fix: handle empty todo title
docs: update local setup guide
ci: add pull request workflow
test: add todo service tests
refactor: simplify todo repository
```

## 7. 推送分支

第一次推送新分支：

```bash
git push -u origin feat/todo-priority
```

后续继续推送：

```bash
git push
```

`-u` 的意思是建立本地分支和远程分支的默认关联。

## 8. 修改最近一次 commit

如果你刚提交完，发现漏了一个小改动：

```bash
git add README.md
git commit --amend
```

如果只想改 commit message：

```bash
git commit --amend -m "feat: add todo priority"
```

注意：如果这个 commit 已经被别人基于它开发，修改历史会影响别人。个人 feature 分支上通常可以这么做，公共分支上不要随便做。

## 9. 拉取别人更新

开发过程中 main 可能已经变化。你可以把 main 的更新合进当前分支。

方式一：merge。

```bash
git fetch origin
git merge origin/main
```

方式二：rebase。

```bash
git fetch origin
git rebase origin/main
```

初学建议：

- 团队没明确要求时，用 merge 更安全。
- 在自己的 feature 分支上，可以学习 rebase。
- 不要 rebase 已经多人共享的公共分支。

## 10. merge 和 rebase 的区别

### merge

merge 会保留分支历史，并产生一个合并提交。

```text
main: A---B---C
           \ 
feature:    D---E

merge 后：

main: A---B---C---M
           \     /
feature:    D---E
```

特点：

- 历史真实。
- 不改写已有 commit。
- 提交图可能更复杂。

### rebase

rebase 会把你的分支提交“搬到”最新 main 后面。

```text
main: A---B---C
           \
feature:    D---E

rebase 后：

main: A---B---C
             \
feature:      D'---E'
```

特点：

- 历史更线性。
- 会改写 commit hash。
- 对共享分支要谨慎。

## 11. 解决冲突

冲突常见于多人修改同一段代码。

当 Git 提示 conflict：

```bash
git status
```

打开冲突文件，你会看到类似：

```text
<<<<<<< HEAD
old code
=======
new code
>>>>>>> feature
```

你需要手动选择或整合代码，然后：

```bash
git add conflicted-file.go
git commit
```

如果是在 rebase 中解决冲突：

```bash
git add conflicted-file.go
git rebase --continue
```

想放弃本次 merge：

```bash
git merge --abort
```

想放弃本次 rebase：

```bash
git rebase --abort
```

## 12. 常用排查命令

查看简洁历史：

```bash
git log --oneline --graph --decorate --all
```

查看某个文件是谁改的：

```bash
git blame internal/handler/todo.go
```

查看某个 commit 的内容：

```bash
git show 8f3a2c1
```

查看最近操作记录：

```bash
git reflog
```

`reflog` 很有用。误操作后，经常可以从这里找回位置。

## 13. 推荐日常流程

```bash
git switch main
git pull origin main
git switch -c feat/some-change

# 修改代码

git status
git diff
git add .
git commit -m "feat: describe change"
git push -u origin feat/some-change
```

然后创建 PR。

## 14. 小练习

在你的练习仓库里完成：

1. 从 main 创建 `feat/hello-api` 分支。
2. 新增或修改一个文件。
3. 查看 `git status` 和 `git diff`。
4. 提交 commit：`feat: add hello api note`。
5. 推送到远程仓库。
6. 查看远程平台上是否出现新分支。

## 15. 本节小结

你现在应该能完成：

- 更新 main。
- 创建功能分支。
- 提交清晰 commit。
- 推送远程分支。
- 理解 merge 和 rebase 的区别。
- 解决最基本的冲突。

