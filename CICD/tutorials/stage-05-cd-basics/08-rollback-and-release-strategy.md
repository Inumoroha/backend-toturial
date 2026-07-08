# 08：回滚与发布策略

## 1. 本节目标

发布一定要考虑失败。

这一节学习：

- 镜像回滚。
- 配置回滚。
- release tag。
- hotfix。
- rollback workflow。
- 数据库变更风险。

## 2. 最小回滚策略

本阶段最小回滚方式：

```text
把 APP_IMAGE 改回上一个稳定镜像
-> docker compose pull api
-> docker compose up -d api
-> 健康检查
-> 冒烟测试
```

前提：

- 你知道上一个稳定镜像。
- registry 中还保留该镜像。
- 数据库变更兼容旧版本。

## 3. rollback.sh 示例

```bash
#!/usr/bin/env bash
set -euo pipefail

APP_DIR="${APP_DIR:-/opt/go-cicd-lab}"
cd "$APP_DIR"

if [ ! -f .previous_image ]; then
  echo ".previous_image not found" >&2
  exit 1
fi

IMAGE="$(cat .previous_image)"
echo "Rolling back to $IMAGE"

IMAGE="$IMAGE" ./deploy.sh
```

注意：这个脚本依赖 `deploy.sh`。

## 4. 手动指定镜像回滚

有时你不想用 `.previous_image`，而是指定某个版本：

```bash
IMAGE=ghcr.io/your-name/go-cicd-lab:v0.1.0 ./deploy.sh
```

这是最清晰的回滚方式。

生产回滚时推荐明确写出目标镜像。

## 5. Release 策略

推荐：

```text
main -> staging
tag v* -> production
```

生产发布：

```bash
git tag -a v0.1.0 -m "release v0.1.0"
git push origin v0.1.0
```

生产部署镜像：

```text
ghcr.io/your-name/go-cicd-lab:v0.1.0
```

## 6. Hotfix 策略

生产紧急修复：

```text
确认当前生产版本
-> 创建 fix 分支
-> 修复
-> PR + CI
-> 合并 main
-> 打 patch tag
-> 部署 production
```

例如：

```text
v0.1.0 -> v0.1.1
```

如果 main 已经包含未准备发布的新功能，可以从生产 tag 创建 hotfix 分支，然后把修复合回 main。

## 7. 配置回滚

配置也可能导致事故：

- `DATABASE_URL` 错误。
- feature flag 打开错误。
- Redis 地址错误。
- 日志级别过高。

配置回滚需要：

- 知道上一个配置值。
- 配置变更有记录。
- 重要配置通过 PR 或 secret 变更记录管理。

不要只回滚镜像却忽略配置。

## 8. 数据库变更和回滚

数据库迁移最危险。

镜像回滚容易，数据库回滚不一定容易。

例如：

- 新增字段：通常比较安全。
- 删除字段：旧版本可能无法运行。
- 改字段类型：可能破坏读写。
- 数据修正：可能不可逆。

建议：

```text
先兼容，再切换，最后清理。
```

第 9 篇会专门讲迁移。

## 9. GitHub Actions 手动回滚

可以用 `workflow_dispatch` 输入镜像：

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [staging, production]
        required: true
      image:
        required: true
        description: "Image to deploy or rollback to"
```

回滚就是一次特殊部署：

```text
部署旧镜像。
```

production 回滚同样应该经过 environment 审批，除非团队定义了紧急通道。

## 10. 小练习

1. 部署版本 A。
2. 部署版本 B。
3. 查看 `.previous_image`。
4. 执行：

```bash
IMAGE=<version-A-image> ./deploy.sh
```

5. 检查 `/version` 是否回到版本 A。

## 11. 本节小结

你现在应该理解：

- 回滚的核心是部署上一个稳定镜像。
- 回滚前必须知道当前版本和目标版本。
- 配置也需要回滚策略。
- 数据库变更可能让镜像回滚失效。
- production 回滚仍然应该可审计。

