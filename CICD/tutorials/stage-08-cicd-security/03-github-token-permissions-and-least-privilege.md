# 03：GITHUB_TOKEN、权限与最小权限

## 1. 本节目标

GitHub Actions 会自动提供 `GITHUB_TOKEN`。

这一节学习：

- `GITHUB_TOKEN` 是什么。
- workflow 级和 job 级 permissions。
- 常见权限配置。
- 如何把权限收紧。

## 2. GITHUB_TOKEN 是什么

`GITHUB_TOKEN` 是 GitHub 为每个 workflow job 自动创建的 token。

它可以访问当前仓库相关 API。

问题是：

```text
如果权限给太大，workflow 中的恶意代码也会继承这些权限。
```

所以必须显式设置最小权限。

## 3. 只读 CI

普通 PR CI：

```yaml
permissions:
  contents: read
```

适合：

- checkout。
- test。
- lint。
- build check。

不需要：

- packages: write。
- deployments: write。
- contents: write。
- id-token: write。

## 4. 推镜像

推 GHCR 镜像：

```yaml
permissions:
  contents: read
  packages: write
```

只给 image build/push workflow。

不要给普通 PR test job。

## 5. OIDC

使用 OIDC：

```yaml
permissions:
  contents: read
  id-token: write
```

`id-token: write` 只是允许 workflow 请求 OIDC token，不等于自动拥有云权限。

云端信任策略决定这个 token 能换什么权限。

## 6. Artifact Attestation

生成 attestation：

```yaml
permissions:
  contents: read
  id-token: write
  attestations: write
```

如果推容器镜像并关联 attestation：

```yaml
permissions:
  contents: read
  id-token: write
  attestations: write
  packages: write
```

## 7. Job 级权限

推荐在 workflow 级设置默认只读：

```yaml
permissions:
  contents: read
```

只在需要的 job 提权：

```yaml
jobs:
  publish:
    permissions:
      contents: read
      packages: write
```

这样 test job 即使被注入，也没有 publish 权限。

## 8. 避免 write-all

不要随便：

```yaml
permissions: write-all
```

也不要因为某个 Action 报权限错误就直接给全部写权限。

正确做法：

1. 看 Action 文档需要哪些权限。
2. 只加对应权限。
3. 尽量 job 级添加。

## 9. 审查现有 workflow

用表格审查：

| Workflow | Job | 当前权限 | 实际需要 | 是否过宽 |
| --- | --- | --- | --- | --- |
| ci.yml | test | contents: read | contents: read | 否 |
| image.yml | build | packages: write | packages: write | 否 |
| deploy.yml | deploy | id-token: write | 取决于 OIDC | 待审查 |

## 10. 小练习

1. 打开所有 `.github/workflows/*.yml`。
2. 为每个 workflow 添加顶层：

```yaml
permissions:
  contents: read
```

3. 对推镜像 job 单独增加 `packages: write`。
4. 对 OIDC job 单独增加 `id-token: write`。
5. 确认 PR CI 不具备写权限。

## 11. 本节小结

你现在应该理解：

- `GITHUB_TOKEN` 权限必须显式最小化。
- PR CI 通常只需要 `contents: read`。
- 推镜像才需要 `packages: write`。
- OIDC 才需要 `id-token: write`。
- job 级权限比 workflow 全局提权更安全。

