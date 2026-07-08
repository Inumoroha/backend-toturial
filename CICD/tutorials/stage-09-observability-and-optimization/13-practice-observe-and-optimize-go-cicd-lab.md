# 13：实战：观测并优化 go-cicd-lab

## 1. 本节目标

这一节把第 9 阶段内容落到 `go-cicd-lab`。

你要完成：

```text
docs/cicd-observability.md
docs/pipeline-optimization-report.md
docs/delivery-report-template.md
.github/workflows/ci.yml
.github/workflows/image.yml
```

目标是形成一份可解释的优化报告，而不是只改 YAML。

## 2. 第一步：建立可观测文档

创建：

```text
docs/cicd-observability.md
```

内容：

```markdown
# CI/CD Observability

## Key Questions

- How long does PR CI take?
- Which job fails most often?
- How long does image build take?
- How do we verify staging and production?

## Key Metrics

| Metric | Source | Target |
| --- | --- | --- |
| PR CI p50 | GitHub Actions | TBD |
| PR CI p95 | GitHub Actions | TBD |
| CI success rate | GitHub Actions | TBD |
| Docker build duration | GitHub Actions | TBD |
| Deployment duration | Argo CD / workflow | TBD |
| Smoke test result | workflow | must pass |

## Release Evidence

- Commit SHA
- Workflow run URL
- Image digest
- Deploy repo commit
- Argo CD application
- Smoke test result
```

## 3. 第二步：导出最近 workflow 数据

创建目录：

```bash
mkdir -p reports
```

导出：

```bash
gh api repos/:owner/:repo/actions/runs \
  -f per_page=100 \
  > reports/workflow-runs.json
```

统计结果：

```bash
jq -r '.workflow_runs[].conclusion' reports/workflow-runs.json | sort | uniq -c
```

记录最近 10 次：

```bash
gh run list \
  --limit 10 \
  --json databaseId,name,event,headBranch,conclusion,createdAt,updatedAt
```

## 4. 第三步：写优化报告

创建：

```text
docs/pipeline-optimization-report.md
```

模板：

```markdown
# Pipeline Optimization Report

## Baseline

Date:

| Metric | Before |
| --- | --- |
| PR CI p50 | |
| PR CI p95 | |
| CI success rate | |
| Slowest job | |
| Docker build duration | |
| Image scan duration | |

## Bottlenecks

1.
2.
3.

## Changes

| Change | Expected Impact | Risk |
| --- | --- | --- |
| Enable Go cache | Faster test/setup | Cache invalidation |
| Enable Docker buildx cache | Faster image build | Cache storage |
| Add concurrency | Reduce wasted runs | Cancel old PR runs |

## Result

| Metric | Before | After | Difference |
| --- | --- | --- | --- |
| PR CI p50 | | | |
| PR CI p95 | | | |
| Docker build duration | | | |
```

## 5. 第四步：优化 ci.yml

建议结构：

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

      - name: Test
        run: |
          mkdir -p reports
          go test -json ./... > reports/go-test.json

      - name: Upload test report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: go-test-report
          path: reports/go-test.json
          retention-days: 14

      - name: Summary
        if: always()
        run: |
          {
            echo "## CI Summary"
            echo ""
            echo "| Item | Value |"
            echo "| --- | --- |"
            echo "| Commit | \`${GITHUB_SHA}\` |"
            echo "| Go | $(go version) |"
          } >> "$GITHUB_STEP_SUMMARY"
```

如果 test、lint、vuln 都很慢，可以拆成多个 job。

## 6. 第五步：优化 image.yml

示例：

```yaml
name: image

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write

concurrency:
  group: image-${{ github.ref }}
  cancel-in-progress: true

jobs:
  image:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

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
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Summary
        run: |
          {
            echo "## Image Summary"
            echo ""
            echo "| Item | Value |"
            echo "| --- | --- |"
            echo "| Image | \`ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}\` |"
            echo "| Commit | \`${GITHUB_SHA}\` |"
          } >> "$GITHUB_STEP_SUMMARY"
```

如果你沿用了第 8 阶段的签名、SBOM、attestation，继续保留，不要为了速度删除安全门禁。

## 7. 第六步：增加 smoke test

创建：

```text
scripts/smoke-test.sh
```

内容：

```bash
#!/usr/bin/env bash
set -euo pipefail

base_url="${BASE_URL:?BASE_URL is required}"

curl -fsS "$base_url/healthz"
curl -fsS "$base_url/readyz"
curl -fsS "$base_url/version"
```

workflow：

```yaml
- name: Smoke test
  env:
    BASE_URL: ${{ vars.STAGING_BASE_URL }}
  run: bash scripts/smoke-test.sh
```

## 8. 第七步：给 Go 服务增加版本和 metrics

要求：

- `/healthz`。
- `/readyz`。
- `/version` 输出 commit。
- `/metrics` 输出 Prometheus metrics。

构建时注入 commit：

```bash
go build -ldflags "-X main.commit=$GITHUB_SHA" -o app ./cmd/api
```

发布后验证：

```bash
curl -fsS "$BASE_URL/version"
curl -fsS "$BASE_URL/metrics" | head
```

## 9. 第八步：交付周报模板

创建：

```text
docs/delivery-report-template.md
```

记录：

```markdown
# Delivery Report

Period:

## CI Health

- PR CI p50:
- PR CI p95:
- CI success rate:
- Most failed job:

## Deployment

- Production deployments:
- Change failure rate:
- Mean restore time:

## Improvements

- Completed:
- Next:
- Risks:
```

## 10. 验收标准

完成后你应该能展示：

- 最近 100 次 workflow 的基础统计。
- 一份优化前基线。
- 至少 2 个具体优化改动。
- 优化后对比数据。
- CI summary 输出关键信息。
- 测试报告 artifact。
- Docker buildx cache 配置。
- PR concurrency 配置。
- smoke test 脚本。
- Go 服务 `/metrics` 和 `/version`。
- 一份交付周报模板。

## 11. 本节小结

你现在完成的是工程化优化闭环：

```text
收集数据
建立基线
提出假设
实施优化
验证效果
形成报告
持续改进
```

这比“凭感觉改 YAML”成熟得多。

