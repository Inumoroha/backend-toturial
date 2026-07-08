# 08：SBOM、Provenance 与 SLSA

## 1. 本节目标

当一个镜像进入 production，你应该能回答三个问题：

```text
里面有什么？
从哪里构建出来？
构建过程是否可信？
```

这一节学习：

- SBOM 是什么。
- provenance 是什么。
- artifact attestation 是什么。
- SLSA 解决什么问题。
- 如何在 CI 中生成 SBOM 和 provenance。

## 2. SBOM 是什么

SBOM 是 Software Bill of Materials，软件物料清单。

它描述一个制品里包含哪些组件，例如：

```text
Go module
系统包
容器基础镜像层
二进制依赖
许可证信息
版本信息
```

你可以把它理解为：

```text
软件制品的配料表。
```

常见格式：

- SPDX。
- CycloneDX。

SBOM 本身不等于安全。

它的价值是：

- 漏洞爆发时能快速判断是否受影响。
- 给审计和客户提供依赖透明度。
- 和镜像 digest、签名、provenance 一起形成供应链证据。

## 3. 使用 Trivy 生成 SBOM

为镜像生成 CycloneDX SBOM：

```bash
trivy image \
  --format cyclonedx \
  --output sbom.cdx.json \
  ghcr.io/your-org/go-cicd-lab:sha-xxxx
```

生成 SPDX JSON：

```bash
trivy image \
  --format spdx-json \
  --output sbom.spdx.json \
  ghcr.io/your-org/go-cicd-lab:sha-xxxx
```

扫描 SBOM：

```bash
trivy sbom sbom.cdx.json
```

建议保存：

```text
镜像 digest
SBOM 文件
构建日志
provenance/attestation
```

## 4. Provenance 是什么

provenance 可以理解为构建来源证明。

它通常回答：

```text
哪个仓库？
哪个 commit？
哪个 workflow？
哪个 runner？
什么时候构建？
构建产物是什么 digest？
构建命令或构建系统是什么？
```

SBOM 说“里面有什么”。

provenance 说“它怎么来的”。

两者经常一起使用，但不是同一个东西。

## 5. Artifact Attestation 是什么

attestation 是对某个制品的可验证声明。

例如：

```text
我声明这个镜像 digest 是由 github.com/your-org/go-cicd-lab 的 main 分支在某次 workflow 中构建的。
```

这个声明会被签名，并关联到制品 digest。

GitHub Artifact Attestations 可以在 GitHub Actions 中生成 provenance 或 SBOM attestation。

根据 GitHub 官方文档，生成 attestation 通常需要：

```yaml
permissions:
  contents: read
  id-token: write
  attestations: write
```

如果还要推送 GHCR 镜像，通常还需要：

```yaml
permissions:
  packages: write
```

## 6. 为容器镜像生成 provenance

示例：

```yaml
name: image

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write
  id-token: write
  attestations: write

jobs:
  build:
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

      - name: Generate provenance attestation
        uses: actions/attest-build-provenance@v2
        with:
          subject-name: ghcr.io/${{ github.repository }}
          subject-digest: ${{ steps.build.outputs.digest }}
          push-to-registry: true
```

如果官方推荐 action 版本变化，以 GitHub 文档为准。

## 7. 为 SBOM 生成 attestation

先生成 SBOM：

```yaml
      - name: Generate SBOM
        run: |
          trivy image \
            --format cyclonedx \
            --output sbom.cdx.json \
            ghcr.io/${{ github.repository }}:${{ github.sha }}
```

再生成 SBOM attestation：

```yaml
      - name: Generate SBOM attestation
        uses: actions/attest-sbom@v2
        with:
          subject-name: ghcr.io/${{ github.repository }}
          subject-digest: ${{ steps.build.outputs.digest }}
          sbom-path: sbom.cdx.json
          push-to-registry: true
```

有些平台会使用统一的 `actions/attest` action 处理 provenance 和 SBOM。写生产 workflow 前，务必按官方文档确认当前推荐写法。

## 8. SLSA 是什么

SLSA 是 Supply-chain Levels for Software Artifacts。

它不是一个具体工具，而是一套供应链安全框架。

你可以先理解为几个方向：

```text
源代码是否受保护？
构建是否可追溯？
构建是否在受控环境中执行？
制品是否有 provenance？
消费者能否验证制品来源？
```

入门阶段不要陷入级别细节，先做到：

- 主分支受保护。
- CI 权限最小化。
- 构建环境可追溯。
- 镜像使用 digest。
- 生成 SBOM。
- 生成 provenance。
- 镜像签名并验证。

## 9. SBOM、provenance、签名的关系

可以这样理解：

```text
SBOM：里面有什么。
provenance：怎么构建出来。
签名：谁对这个 digest 做了可验证声明。
准入策略：不满足要求就不允许部署。
```

单独做其中一个，价值有限。

组合起来后，生产链路会变成：

```text
构建镜像
生成 digest
扫描镜像
生成 SBOM
生成 provenance
签名 digest
部署时验证 digest、签名和来源
```

## 10. 小练习

在 `go-cicd-lab` 中完成：

1. 为镜像生成 `sbom.cdx.json`。
2. 使用 Trivy 扫描 SBOM。
3. 在 CI 中上传 SBOM artifact。
4. 为镜像生成 provenance attestation。
5. 记录镜像 digest，而不是只记录 tag。
6. 用自己的话解释 SBOM 和 provenance 的区别。

## 11. 本节小结

你现在应该理解：

- SBOM 是软件制品的组件清单。
- provenance 是构建来源和过程证明。
- attestation 是可验证声明。
- SLSA 是供应链安全成熟度框架。
- 对生产镜像，digest、SBOM、provenance、签名应逐步成为标准输出。

