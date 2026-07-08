# 09：镜像签名与验证

## 1. 本节目标

镜像扫描告诉你“有没有发现漏洞”。

镜像签名回答另一个问题：

```text
这个镜像 digest 是否真的是由可信身份构建和发布的？
```

这一节学习：

- 为什么要签名镜像。
- Cosign keyless signing 是什么。
- 如何在 CI 中签名镜像。
- 如何验证镜像签名。
- 部署时如何使用签名验证。

## 2. 为什么签名的是 digest，不是 tag

tag 是名字：

```text
ghcr.io/your-org/go-cicd-lab:latest
```

digest 是内容地址：

```text
ghcr.io/your-org/go-cicd-lab@sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

tag 可以移动，digest 指向具体内容。

所以签名和验证都应该围绕 digest。

## 3. Cosign keyless signing

传统签名需要长期私钥。

keyless signing 的思路是：

```text
CI 使用 OIDC 获取短期身份。
签名系统根据这个身份签发短期证书。
签名结果和身份绑定。
验证时检查镜像 digest、签名、证书身份和 issuer。
```

在 GitHub Actions 中，通常需要：

```yaml
permissions:
  contents: read
  packages: write
  id-token: write
```

`id-token: write` 用于让 workflow 请求 OIDC token。

## 4. 本地安装 Cosign

安装后查看版本：

```bash
cosign version
```

验证一个镜像的签名时通常会用到：

```bash
cosign verify \
  --certificate-identity "https://github.com/your-org/go-cicd-lab/.github/workflows/image.yml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/your-org/go-cicd-lab@sha256:xxxx
```

真实 identity 会和你的仓库、workflow、ref 有关，请以实际签名证书为准。

## 5. CI 中签名镜像

示例：

```yaml
name: signed-image

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write
  id-token: write

jobs:
  image:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Install cosign
        uses: sigstore/cosign-installer@v3

      - name: Sign image
        env:
          COSIGN_EXPERIMENTAL: "1"
          IMAGE: ghcr.io/${{ github.repository }}
          DIGEST: ${{ steps.build.outputs.digest }}
        run: |
          cosign sign --yes "$IMAGE@$DIGEST"
```

说明：

- `docker/build-push-action` 输出 digest。
- `cosign sign` 对 `image@digest` 签名。
- keyless signing 依赖 OIDC。
- 生产 workflow 建议 pin action 到 commit SHA。

Cosign 的参数可能随版本演进，写生产流水线前要查看 Sigstore 官方文档。

## 6. 验证镜像

在部署前或本地验证：

```bash
cosign verify \
  --certificate-identity-regexp "https://github.com/your-org/go-cicd-lab/.github/workflows/.*@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/your-org/go-cicd-lab@sha256:xxxx
```

验证重点：

```text
镜像 digest 是否匹配。
签名是否有效。
证书 identity 是否来自你的仓库和 workflow。
OIDC issuer 是否是 GitHub Actions。
```

如果只验证“有签名”，但不验证 identity，价值会大幅降低。

## 7. 部署时验证的几种位置

可以在不同位置验证：

```text
CI 发布 job 中验证。
部署仓库 PR 检查中验证。
Argo CD 同步前的检查中验证。
Kubernetes admission controller 中验证。
```

越靠近集群准入，防护越强。

但学习阶段可以先从简单做起：

```text
在部署仓库 PR 或发布 workflow 中验证镜像签名。
验证失败就不允许更新 production values。
```

## 8. Kubernetes 准入策略

更成熟的团队会使用策略控制器：

- Sigstore Policy Controller。
- Kyverno。
- Connaisseur。
- 其他 admission webhook。

目标是：

```text
没有可信签名的镜像不能进入集群。
签名身份不匹配的镜像不能进入 production namespace。
```

示意策略：

```text
只允许来自 ghcr.io/your-org 的镜像。
镜像必须使用 digest。
镜像必须有 Cosign keyless 签名。
签名 identity 必须来自指定 GitHub 仓库 main 分支 workflow。
```

## 9. 常见误区

### 误区 1：签名 tag

应该签名 digest。

### 误区 2：只检查镜像能拉取

能拉取不代表可信。

### 误区 3：只看签名存在

还要验证签名身份。

### 误区 4：CI 有签名，部署时不验证

签名不验证就只是记录。

### 误区 5：长期私钥随便放在 secrets

如果使用密钥式签名，私钥就是高敏感 secret，必须最小权限和轮换。

keyless 能减少长期私钥管理负担。

## 10. 小练习

完成：

1. 在 CI 中获取镜像 digest。
2. 使用 Cosign keyless 签名镜像。
3. 本地或 CI 中验证签名。
4. 写出允许部署到 production 的签名 identity 条件。
5. 修改部署 values，尽量使用 `image.digest` 而不是只使用 tag。

## 11. 本节小结

你现在应该理解：

- 镜像签名应围绕 digest。
- keyless signing 通过 OIDC 减少长期私钥。
- 验证时必须检查 identity 和 issuer。
- 签名只有在部署链路中被验证，才会形成真正门禁。
- 最终目标是让 production 只运行可信来源构建的镜像。

