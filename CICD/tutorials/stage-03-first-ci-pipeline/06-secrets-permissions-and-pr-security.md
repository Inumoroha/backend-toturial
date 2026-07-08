# 06：Secret、权限与 PR 安全

## 1. 本节目标

第 3 阶段虽然不部署，但必须提前建立 CI 安全意识。

这一节学习：

- `GITHUB_TOKEN`。
- `permissions`。
- secrets。
- fork PR 风险。
- `pull_request` 和 `pull_request_target` 的区别。
- 最小权限原则。

## 2. CI 为什么危险

CI runner 可以：

- 读取源码。
- 执行仓库里的脚本。
- 访问 secret。
- 调用 GitHub API。
- 后续阶段还会推镜像、部署环境。

所以 CI 不是“安全的自动脚本”，而是一个需要认真设权的执行环境。

## 3. GITHUB_TOKEN

GitHub Actions 会为 workflow 提供 `GITHUB_TOKEN`。

它可以用于：

- 读取仓库内容。
- 给 PR 写评论。
- 创建 release。
- 推送 package。
- 调用 GitHub API。

具体能做什么，取决于权限配置。

第 3 阶段的 CI 只需要读代码：

```yaml
permissions:
  contents: read
```

## 4. permissions 最小化

推荐每个 workflow 显式写：

```yaml
permissions:
  contents: read
```

如果某个 job 需要更高权限，再单独给 job：

```yaml
jobs:
  comment:
    permissions:
      contents: read
      pull-requests: write
```

不要为了省事给：

```yaml
permissions: write-all
```

除非你非常清楚风险。

## 5. Secrets

GitHub repository secrets 使用方式：

```yaml
env:
  TOKEN: ${{ secrets.SOME_TOKEN }}
```

第 3 阶段的 PR CI 通常不需要任何生产 secret。

如果你的测试需要数据库，可以优先使用：

- CI service container。
- 临时测试数据库。
- Docker Compose。
- 测试专用账号。

不要让 PR job 拿生产数据库密码。

## 6. fork PR 风险

开源仓库或多人协作时，外部贡献者可以从 fork 提 PR。

风险：

```text
PR 中的代码会在 CI 中执行。
恶意代码可能尝试读取环境变量、访问 token、打印 secret。
```

所以原则是：

- PR CI 尽量只读。
- PR CI 不使用生产 secret。
- 对外部 PR 更谨慎。
- 不在日志中打印环境变量。

## 7. pull_request_target 要谨慎

`pull_request_target` 和 `pull_request` 不同。

`pull_request_target` 在目标仓库上下文中运行，可能拥有更高权限。

危险模式：

```yaml
on: pull_request_target

steps:
  - uses: actions/checkout@v6
    with:
      ref: ${{ github.event.pull_request.head.sha }}
  - run: ./scripts/something.sh
```

如果这个脚本来自不受信任 PR，就可能用更高权限执行恶意代码。

初学阶段建议：

```text
只使用 pull_request，不使用 pull_request_target。
```

除非你明确知道它的安全模型。

## 8. 不要打印敏感上下文

不要这样做：

```yaml
- run: env
```

也不要随便打印：

```yaml
- run: echo "${{ toJson(secrets) }}"
```

平台可能会做脱敏，但你不应该依赖脱敏解决所有问题。

## 9. 第 3 阶段推荐安全配置

```yaml
name: ci

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-go@v5
        with:
          go-version: "1.25.x"
          cache: true
      - run: go test ./...
```

没有生产 secret。

没有写权限。

没有部署。

这是一个很好的 CI 起点。

## 10. 后续阶段才需要的权限

第 4 阶段推镜像时，可能需要：

```yaml
permissions:
  contents: read
  packages: write
```

第 5 阶段部署时，可能需要：

- environment secret。
- OIDC。
- 云厂商权限。
- 人工审批。

这些都不应该提前塞进第 3 阶段的 PR CI。

## 11. 小练习

检查你的 `ci.yml`：

1. 是否有 `permissions: contents: read`？
2. PR job 是否使用了任何 secret？
3. 是否有 `pull_request_target`？
4. 是否有打印环境变量的 step？
5. 是否有不必要的写权限？

如果有，写清楚为什么需要。

## 12. 本节小结

你现在应该理解：

- CI 会执行代码，因此它是安全边界的一部分。
- `GITHUB_TOKEN` 权限应该最小化。
- PR CI 不应该使用生产 secret。
- 初学阶段避免 `pull_request_target`。
- 第 3 阶段只做检查，不给部署权限。

