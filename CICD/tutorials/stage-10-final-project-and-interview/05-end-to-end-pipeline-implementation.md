# 05：端到端流水线实现

## 1. 本节目标

这一节把整个链路串起来。

目标是让你能从一次代码修改开始，最终解释：

```text
这次变更如何被检查？
如何被构建成镜像？
如何证明镜像可信？
如何进入 staging？
如何进入 production？
如何回滚？
```

## 2. Workflow 总览

应用仓库建议有：

```text
.github/workflows/ci.yml
.github/workflows/codeql.yml
.github/workflows/image.yml
.github/workflows/update-deploy-repo.yml
```

职责：

| Workflow | 触发 | 职责 |
| --- | --- | --- |
| `ci.yml` | PR、main push | test、lint、govulncheck |
| `codeql.yml` | PR、main、schedule | 代码扫描 |
| `image.yml` | main push、tag、manual | 构建、扫描、SBOM、签名、推送 |
| `update-deploy-repo.yml` | image 成功后或手动 | 更新部署仓库 |

## 3. ci.yml 要点

```yaml
name: ci

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true
      - run: make test

  vuln:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true
      - run: go install golang.org/x/vuln/cmd/govulncheck@latest
      - run: make vuln
```

生产中请按官方文档确认 action 版本，关键 workflow 可 pin 到 commit SHA。

## 4. image.yml 要点

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
  image:
    runs-on: ubuntu-latest
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
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
          tags: ${{ env.IMAGE }}:${{ github.sha }}
          build-args: |
            VERSION=${{ github.sha }}
            COMMIT=${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Image summary
        run: |
          {
            echo "## Image"
            echo ""
            echo "- Image: \`${IMAGE}@${{ steps.build.outputs.digest }}\`"
            echo "- Commit: \`${GITHUB_SHA}\`"
          } >> "$GITHUB_STEP_SUMMARY"
```

如果你已经完成第 8 阶段，应继续加入：

- Trivy 扫描。
- SBOM。
- provenance/attestation。
- Cosign 签名。

## 5. update-deploy-repo.yml 要点

这个 workflow 负责把新镜像写入部署仓库。

staging 可以自动更新：

```yaml
name: update-deploy-repo

on:
  workflow_dispatch:
    inputs:
      environment:
        description: Environment
        required: true
        default: staging
      image_digest:
        description: Image digest
        required: true

permissions:
  contents: read

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout deploy repo
        uses: actions/checkout@v4
        with:
          repository: your-org/go-cicd-lab-deploy
          token: ${{ secrets.DEPLOY_REPO_TOKEN }}
          path: deploy

      - name: Update values
        working-directory: deploy
        run: |
          yq -i '.image.digest = "${{ inputs.image_digest }}"' \
            environments/${{ inputs.environment }}/values.yaml

      - name: Commit change
        working-directory: deploy
        run: |
          git config user.name "github-actions"
          git config user.email "github-actions@github.com"
          git add environments/${{ inputs.environment }}/values.yaml
          git commit -m "Update ${{ inputs.environment }} image digest"
          git push
```

生产中更推荐创建 PR，而不是直接 push。

## 6. Production 发布策略

建议：

```text
staging：自动更新、自动同步、自动 smoke test。
production：创建部署仓库 PR、CODEOWNERS review、手动 sync 或受控自动 sync。
```

production PR 必须包含：

- 源 commit。
- image digest。
- workflow run。
- SBOM/provenance。
- 签名验证结果。
- 回滚方式。

## 7. 端到端验收流程

执行一次：

```text
1. 创建 feature branch。
2. 提交一个 API 变更。
3. 创建 PR。
4. 确认 ci/codeql 通过。
5. 合并 main。
6. image workflow 构建并推送镜像。
7. 更新 staging 部署仓库 values。
8. Argo CD 同步 staging。
9. smoke test 通过。
10. 创建 production PR。
11. 审批并合并。
12. Argo CD 同步 production。
13. 观察 metrics。
```

## 8. 小练习

写一份：

```text
docs/e2e-release-walkthrough.md
```

内容：

```markdown
# End-to-End Release Walkthrough

## Change

## PR Checks

## Image

## Staging

## Production

## Verification

## Rollback
```

## 9. 本节小结

你现在应该能把前面学的所有环节串起来。

端到端流水线的重点不是 YAML 多复杂，而是：

- 每一步职责清楚。
- 每一步权限合理。
- 每一步有证据。
- 失败时知道在哪里查。

