# 12：实战：加固 go-cicd-lab 流水线

## 1. 本节目标

这一节把前面的安全知识落到一个 Go 后端项目上。

目标不是一次做到企业级完美，而是完成一条可解释、可审查、可逐步加固的安全基线。

你将输出：

```text
.github/workflows/ci.yml
.github/workflows/codeql.yml
.github/workflows/image.yml
.github/dependabot.yml
docs/security.md
docs/security-incident-runbook.md
```

## 2. 最终目标链路

建议链路：

```text
PR
  -> Go test/lint/govulncheck
  -> CodeQL
  -> Docker build
  -> Trivy scan

main
  -> Build image
  -> Push image
  -> Generate SBOM
  -> Generate provenance
  -> Sign image
  -> Verify signature
  -> Update staging deploy repo

production
  -> Deploy repo PR
  -> CODEOWNERS review
  -> Verify image digest/signature
  -> Argo CD sync
```

## 3. 加固 ci.yml

示例：

```yaml
name: ci

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true

      - name: Download modules
        run: go mod download

      - name: Test
        run: go test -race ./...

      - name: Install govulncheck
        run: go install golang.org/x/vuln/cmd/govulncheck@latest

      - name: Govulncheck
        run: govulncheck ./...
```

关键点：

- 顶层 `permissions: contents: read`。
- 不在 PR test job 暴露部署 secret。
- 先让源码质量门禁稳定。

如果你已经使用 `golangci-lint`，可以继续保留。

## 4. 增加 CodeQL

创建：

```text
.github/workflows/codeql.yml
```

示例：

```yaml
name: codeql

on:
  pull_request:
  push:
    branches: [main]
  schedule:
    - cron: "0 2 * * 1"

permissions:
  contents: read
  security-events: write

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: github/codeql-action/init@v3
        with:
          languages: go

      - uses: github/codeql-action/analyze@v3
```

## 5. 加固 image.yml

下面示例把镜像构建、扫描、SBOM、provenance 和签名放在同一个 workflow。

实际生产中可以拆分为多个 job。

```yaml
name: image

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  packages: write
  id-token: write
  attestations: write

env:
  IMAGE: ghcr.io/${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      digest: ${{ steps.build.outputs.digest }}
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
          tags: |
            ${{ env.IMAGE }}:${{ github.sha }}
            ${{ env.IMAGE }}:main
          labels: |
            org.opencontainers.image.source=${{ github.server_url }}/${{ github.repository }}
            org.opencontainers.image.revision=${{ github.sha }}

      - name: Scan image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE }}@${{ steps.build.outputs.digest }}
          format: table
          severity: HIGH,CRITICAL
          exit-code: "1"

      - name: Generate SBOM
        run: |
          trivy image \
            --format cyclonedx \
            --output sbom.cdx.json \
            "${IMAGE}@${{ steps.build.outputs.digest }}"

      - name: Upload SBOM artifact
        uses: actions/upload-artifact@v4
        with:
          name: sbom-cyclonedx
          path: sbom.cdx.json

      - name: Generate provenance attestation
        uses: actions/attest-build-provenance@v2
        with:
          subject-name: ${{ env.IMAGE }}
          subject-digest: ${{ steps.build.outputs.digest }}
          push-to-registry: true

      - name: Generate SBOM attestation
        uses: actions/attest-sbom@v2
        with:
          subject-name: ${{ env.IMAGE }}
          subject-digest: ${{ steps.build.outputs.digest }}
          sbom-path: sbom.cdx.json
          push-to-registry: true

      - name: Install cosign
        uses: sigstore/cosign-installer@v3

      - name: Sign image
        run: cosign sign --yes "${IMAGE}@${{ steps.build.outputs.digest }}"

      - name: Verify image signature
        run: |
          cosign verify \
            --certificate-identity-regexp "${{ github.server_url }}/${{ github.repository }}/.github/workflows/image.yml@refs/heads/main" \
            --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
            "${IMAGE}@${{ steps.build.outputs.digest }}"
```

说明：

- `packages: write` 用于推送 GHCR 镜像。
- `id-token: write` 用于 OIDC/keyless signing/attestation。
- `attestations: write` 用于 GitHub Artifact Attestations。
- `security-events: write` 只有上传 SARIF 到 code scanning 时才需要。
- 关键生产 workflow 建议将第三方 actions pin 到 commit SHA。

## 6. 增加 Dependabot

创建：

```text
.github/dependabot.yml
```

内容：

```yaml
version: 2
updates:
  - package-ecosystem: gomod
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 5
    groups:
      go-minor-and-patch:
        update-types:
          - minor
          - patch

  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 5
```

Dependabot PR 合并前必须经过测试和 review。

## 7. 增加安全文档

创建：

```text
docs/security.md
```

模板：

```markdown
# Security

## Supported Branches

Only `main` is supported for production releases.

## CI/CD Security Baseline

- GitHub Actions permissions are minimized.
- Production secrets are protected by environments.
- Go dependencies are checked with `govulncheck`.
- Container images are scanned before release.
- Images are signed by CI using keyless signing.
- SBOM and provenance are generated for release images.

## Secrets

- Secrets must not be committed to Git.
- Secrets must not be written into container images.
- Leaked secrets must be revoked and rotated immediately.

## Reporting

Report security issues to: security@example.com

## Release Evidence

For each production release, record:

- Commit SHA
- Workflow run URL
- Image digest
- SBOM artifact
- Provenance/attestation
- Signature verification result
```

## 8. 增加事件响应 Runbook

创建：

```text
docs/security-incident-runbook.md
```

模板：

```markdown
# Security Incident Runbook

## Secret Leak

1. Revoke or rotate the leaked secret.
2. Disable affected workflows if needed.
3. Check workflow logs, artifacts, image layers, and Git history.
4. Rebuild and redeploy clean artifacts.
5. Add prevention controls.

## Suspicious Image

1. Identify image tag and digest.
2. Verify signature and provenance.
3. Stop deployment or roll back to a trusted digest.
4. Check registry push logs and workflow run logs.
5. Rotate credentials if registry access may be compromised.

## Unexpected Production Deployment

1. Identify deploy repo commit.
2. Check Argo CD application history.
3. Revert the deploy repo commit if needed.
4. Confirm service health.
5. Review approval, CODEOWNERS, and branch protection.
```

## 9. 本地验证命令

建议记录一组常用命令：

```bash
go test ./...
govulncheck ./...
docker build -t go-cicd-lab:dev .
trivy image --severity HIGH,CRITICAL go-cicd-lab:dev
trivy image --format cyclonedx --output sbom.cdx.json go-cicd-lab:dev
trivy sbom sbom.cdx.json
```

查看镜像 digest：

```bash
docker buildx imagetools inspect ghcr.io/your-org/go-cicd-lab:main
```

验证签名：

```bash
cosign verify \
  --certificate-identity-regexp "https://github.com/your-org/go-cicd-lab/.github/workflows/.*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/your-org/go-cicd-lab@sha256:xxxx
```

## 10. 验收标准

完成后，你应该能提供：

- `ci.yml` 只读权限运行 Go 检查。
- `codeql.yml` 输出代码扫描结果。
- `image.yml` 构建并推送镜像。
- 镜像扫描阻断 `CRITICAL` 漏洞。
- CI 生成 SBOM。
- CI 生成 provenance 或 artifact attestation。
- CI 使用 keyless signing 签名镜像。
- 有文档说明 secret 泄露响应流程。
- 有文档说明 production 发布证据链。

## 11. 本节小结

你现在把安全控制点串成了一条真实链路：

```text
源码检查
依赖扫描
镜像扫描
SBOM
provenance
签名
验证
部署审查
事件响应
```

这就是 Go 后端工程师在 CI/CD 安全阶段最重要的实战能力。
