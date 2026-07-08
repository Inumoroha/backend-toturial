# 05：Tag、Release 与版本号

## 1. 为什么发布需要版本

生产发布一定要能回答：

```text
现在生产跑的是哪一版？
这一版来自哪个 commit？
如果失败，应该回滚到哪一版？
```

如果你只说“生产跑的是最新代码”，这是不够的。

CI/CD 中，tag 和 release 就是用来表达正式版本的重要工具。

## 2. Git tag 是什么

Tag 是指向某个 commit 的名字。

例如：

```text
v0.1.0 -> commit 8f3a2c1
```

创建轻量 tag：

```bash
git tag v0.1.0
```

推送 tag：

```bash
git push origin v0.1.0
```

查看 tag：

```bash
git tag
```

查看某个 tag 指向的内容：

```bash
git show v0.1.0
```

## 3. Annotated tag

Annotated tag 带有作者、时间和说明，更适合正式发布。

创建：

```bash
git tag -a v0.1.0 -m "release v0.1.0"
git push origin v0.1.0
```

学习项目中，正式发布推荐使用 annotated tag。

## 4. 删除 tag

删除本地 tag：

```bash
git tag -d v0.1.0
```

删除远程 tag：

```bash
git push origin :refs/tags/v0.1.0
```

注意：已经用于生产发布的 tag 不要随便删除或移动。发布版本应该尽量不可变。

## 5. Release 是什么

在 GitHub/GitLab 中，Release 通常是基于 tag 创建的一份发布说明。

Release 可以包含：

- 版本号。
- 变更说明。
- 二进制附件。
- Docker 镜像地址。
- 迁移说明。
- 回滚说明。

Tag 更偏 Git 对象，Release 更偏面向人阅读的发布记录。

## 6. 语义化版本

语义化版本常见格式：

```text
MAJOR.MINOR.PATCH
```

例如：

```text
v1.4.2
```

含义：

| 位置 | 含义 | 示例 |
| --- | --- | --- |
| MAJOR | 不兼容变更 | `v2.0.0` |
| MINOR | 向后兼容的新功能 | `v1.5.0` |
| PATCH | 向后兼容的 bug 修复 | `v1.5.1` |

初学阶段可以这样用：

- 新功能：增加 MINOR。
- bug 修复：增加 PATCH。
- 破坏兼容：增加 MAJOR。

## 7. Conventional Commits 和版本

Conventional Commits 可以帮助生成 changelog。

常见类型：

```text
feat: 新功能
fix: 修复
docs: 文档
test: 测试
ci: CI/CD
chore: 杂项
refactor: 重构
```

示例：

```text
feat: add todo priority
fix: reject empty todo title
ci: add pull request checks
docs: update deployment guide
```

以后你可以用工具根据 commit 自动生成 release note。

## 8. tag 如何触发 CI/CD

常见规则：

```text
push tag v* -> 构建正式版本 -> 推送镜像 -> 创建 release -> 部署 production
```

示意：

```text
git tag -a v0.1.0 -m "release v0.1.0"
git push origin v0.1.0
```

触发：

```text
release pipeline
-> run tests
-> build image
-> tag image v0.1.0
-> push image
-> wait approval
-> deploy production
```

## 9. 镜像 tag 和 Git tag 的关系

Git tag：

```text
v0.1.0
```

Docker image tag：

```text
ghcr.io/your-name/go-cicd-lab:v0.1.0
ghcr.io/your-name/go-cicd-lab:8f3a2c1
```

推荐正式发布同时打两个镜像标签：

```text
版本号 tag：方便人理解。
commit SHA tag：方便精确追溯。
```

## 10. 发布说明模板

```markdown
# v0.1.0

## 新增

- 

## 修复

- 

## 变更

- 

## 数据库迁移

- 是否包含 migration：
- 是否兼容旧版本：

## 镜像

- `ghcr.io/your-name/go-cicd-lab:v0.1.0`
- `ghcr.io/your-name/go-cicd-lab:<commit-sha>`

## 回滚

- 回滚到：
- 回滚验证：
```

## 11. 小练习

在练习仓库里执行：

```bash
git switch main
git pull origin main
git tag -a v0.1.0 -m "release v0.1.0"
git show v0.1.0
git push origin v0.1.0
```

然后在远程平台查看 tag 是否出现。

如果你还不想保留这个 tag，可以删除：

```bash
git tag -d v0.1.0
git push origin :refs/tags/v0.1.0
```

## 12. 本节小结

你现在应该理解：

- tag 是正式版本的重要锚点。
- release 是面向团队和用户的发布记录。
- 语义化版本帮助团队理解变更影响。
- tag 可以触发正式发布流水线。
- 生产版本应该可追溯到 Git commit 和镜像 tag。

