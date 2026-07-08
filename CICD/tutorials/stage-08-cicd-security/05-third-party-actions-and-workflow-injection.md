# 05：第三方 Actions 与 Workflow 注入

## 1. 本节目标

workflow 会执行代码。

第三方 Action、脚本输入、PR 内容都可能成为攻击入口。

这一节学习：

- pin Actions。
- 限制 Actions 来源。
- 防止脚本注入。
- 谨慎处理 PR 输入。

## 2. 第三方 Action 风险

写法：

```yaml
- uses: some-owner/some-action@main
```

风险：

```text
main 分支变化后，你的 workflow 执行内容也变化。
```

即使使用 tag：

```yaml
- uses: some-owner/some-action@v1
```

tag 理论上也可能被移动。

## 3. 生产建议：pin 到 commit SHA

更安全：

```yaml
- uses: some-owner/some-action@0123456789abcdef0123456789abcdef01234567
```

缺点：

- 可读性差。
- 更新需要工具或人工审查。

建议：

- 学习阶段可以用版本 tag。
- 生产关键 workflow pin 到 SHA。
- 使用 Dependabot 或自动化工具更新 Actions。

## 4. 限制允许的 Actions

组织或仓库可以设置：

- 只允许 GitHub 官方 Actions。
- 只允许 verified creator。
- 只允许组织内 Actions。
- 禁止不可信来源。

这能降低供应链风险。

## 5. Script Injection

危险写法：

```yaml
- run: echo "${{ github.event.pull_request.title }}"
```

如果 PR title 包含 shell 特殊字符，可能造成脚本注入。

更安全：

```yaml
- env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: |
    printf '%s\n' "$PR_TITLE"
```

原则：

```text
把不可信上下文放到环境变量里，并用引号处理。
```

## 6. 不可信输入来源

包括：

- PR title。
- PR body。
- issue comment。
- branch name。
- commit message。
- workflow_dispatch input。
- artifact 内容。
- 外部 API 返回。

不要把这些直接拼进 shell 命令。

## 7. pull_request_target 风险

`pull_request_target` 在目标仓库上下文运行，可能能访问更多权限和 secrets。

危险组合：

```yaml
on: pull_request_target
steps:
  - uses: actions/checkout@...
    with:
      ref: ${{ github.event.pull_request.head.sha }}
  - run: ./script-from-pr.sh
```

这可能用高权限执行 PR 里的代码。

初学阶段：

```text
避免使用 pull_request_target。
```

确实需要时，必须严格审查。

## 8. Artifact 污染

一个 job 上传 artifact，另一个高权限 job 下载并执行 artifact，也可能危险。

原则：

- 不执行来自低信任 job 的 artifact。
- 不让 PR job 生成的脚本进入 production job。
- artifact 用于报告和制品，不随便当可执行输入。

## 9. 小练习

检查 workflow：

1. 是否使用 `@main`、`@master`？
2. 是否使用未固定的第三方 Action？
3. 是否直接把 PR title/body 拼进 shell？
4. 是否使用 `pull_request_target`？
5. 是否执行从 artifact 下载的脚本？

## 10. 本节小结

你现在应该理解：

- 第三方 Action 是供应链依赖。
- 生产关键 workflow 应 pin 到 commit SHA。
- PR/issue/dispatch 输入都不可信。
- `pull_request_target` 很强，也很危险。
- 不要执行低信任来源的 artifact。

