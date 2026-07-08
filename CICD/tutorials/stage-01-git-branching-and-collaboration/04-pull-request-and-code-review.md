# 04：Pull Request 与代码评审

## 1. PR/MR 是什么

Pull Request，简称 PR。GitLab 中通常叫 Merge Request，简称 MR。

它表示：

```text
我希望把这个分支的修改合并到目标分支，请团队检查。
```

在 CI/CD 中，PR 是非常重要的质量入口。

PR 通常会触发：

- 自动测试。
- lint。
- build。
- 安全检查。
- 代码评审。
- 分支保护规则。

## 2. PR 的基本流程

```text
创建 feature 分支
-> 提交 commit
-> push 到远程
-> 创建 PR
-> 自动 CI 运行
-> Reviewer 评审
-> 修改代码
-> CI 再次运行
-> 审批通过
-> 合并 main
```

PR 不是形式。它是代码进入主干前的最后一道协作门。

## 3. 好 PR 的标准

一个好的 PR 应该：

- 范围小。
- 目标清楚。
- 描述完整。
- commit 信息清晰。
- 测试结果明确。
- 不混入无关重构。
- 不提交密钥和本地配置。

坏 PR 常见问题：

- 一次改 50 个文件，但目的不清楚。
- 修 bug 时顺手大重构。
- 描述只有“update”。
- CI 失败还要求合并。
- 把 `.env`、私钥、临时日志提交进来。

## 4. PR 描述模板

可以使用这样的模板：

```markdown
## 变更内容

- 

## 为什么需要这个变更

- 

## 如何验证

- [ ] 本地执行 `go test ./...`
- [ ] 本地执行 `go vet ./...`
- [ ] 已验证核心接口

## 风险点

- 

## 回滚方式

- 
```

如果是后端接口变更，可以补充：

```markdown
## API 变化

- 新增：
- 修改：
- 删除：

## 数据库变化

- 是否包含 migration：
- 是否向后兼容：
```

## 5. Reviewer 应该看什么

CI 能自动发现一部分问题，但不是全部。

Reviewer 应该关注：

- 需求是否被正确实现。
- 接口设计是否清晰。
- 错误处理是否完整。
- 日志是否有价值。
- 数据库变更是否安全。
- 是否影响兼容性。
- 是否有安全风险。
- 是否有性能风险。
- 测试是否覆盖关键路径。
- 是否存在无关改动。

对 Go 项目还可以关注：

- `context.Context` 是否正确传递。
- goroutine 是否可能泄漏。
- error 是否包装了足够上下文。
- handler、service、repository 职责是否清楚。
- 是否引入全局状态导致测试困难。

## 6. Author 应该如何处理评审意见

收到评论后：

- 不要急着防御。
- 先确认对方指出的是问题、建议还是疑问。
- 能改就改。
- 不同意时说明原因。
- 修改后回复评论。

常见回复：

```text
已修复，并补充了测试。
这里保留当前实现，因为 xxx，后续如果 yyy 再调整。
这个问题我拆到单独 PR 处理，避免本次变更范围过大。
```

## 7. PR 上的 CI 状态

PR 页面通常会显示检查状态：

```text
lint: passed
test: passed
build: failed
```

合并规则可以要求：

```text
所有 required checks 通过后才允许 merge
```

这就是第 0 阶段说的 gate。

## 8. 合并策略

常见合并策略：

### Merge Commit

保留完整分支历史，产生一个 merge commit。

适合：

- 想保留真实协作历史。
- 团队不强求线性历史。

### Squash Merge

把 PR 中多个 commit 压成一个 commit 合进 main。

适合：

- PR 内 commit 比较碎。
- 希望 main 历史更干净。
- 团队习惯一个 PR 对应一个变更。

### Rebase Merge

把 PR commit 按顺序放到 main 顶部。

适合：

- 团队要求线性历史。
- 开发者对 rebase 比较熟悉。

学习项目推荐：

```text
Squash Merge
```

原因是 main 历史清晰，一个 PR 对应一个完整变更。

## 9. Draft PR

Draft PR 表示“我还没准备好合并，但希望提前让 CI 跑起来或让别人看方向”。

适合：

- 提前验证 CI。
- 让同事早期看设计。
- 大功能拆分时展示进度。

Draft PR 也可以帮助你养成小步提交习惯。

## 10. PR 大小控制

推荐：

```text
一个 PR 解决一个问题。
能在 15 到 30 分钟内评审完最好。
```

如果一个 PR 太大，可以拆成：

```text
PR 1: 添加数据结构和 migration
PR 2: 添加 service/repository
PR 3: 添加 handler 和 API
PR 4: 添加 CI/CD 配置或文档
```

小 PR 更容易通过 CI，更容易 review，也更容易回滚。

## 11. 小练习

请为下面需求写一份 PR 描述：

```text
给 todo API 增加 priority 字段，并支持创建 todo 时传入 priority。
```

至少包含：

- 变更内容。
- 如何验证。
- 数据库变化。
- 风险点。
- 回滚方式。

## 12. 本节小结

你现在应该理解：

- PR/MR 是代码进入主干前的协作门禁。
- CI 和 code review 互补。
- 好 PR 应该小、清晰、可验证。
- 合并策略会影响 main 历史。
- 学习项目推荐使用 Squash Merge。

