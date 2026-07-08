# 07：镜像扫描、SBOM 与基础加固

## 1. 本节目标

镜像构建成功不代表可以放心发布。

这一节学习：

- 镜像漏洞扫描。
- SBOM。
- 基础镜像加固。
- 在 CI 中阻断高危漏洞。

## 2. 镜像扫描解决什么问题

镜像可能包含：

- 有漏洞的基础镜像包。
- 有漏洞的 Go 依赖。
- 不必要的工具。
- 高风险配置。

镜像扫描可以发现已知漏洞，但不能证明镜像绝对安全。

它是基础门禁，不是安全终点。

## 3. 使用 Trivy 扫描本地镜像

安装 Trivy 后：

```bash
trivy image go-cicd-lab:local
```

只关注 HIGH 和 CRITICAL，并让命令失败：

```bash
trivy image --severity HIGH,CRITICAL --exit-code 1 go-cicd-lab:local
```

如果你不想本地安装，也可以用容器运行，具体命令按你的 Docker 环境和 Trivy 官方文档调整。

## 4. 在 GitHub Actions 中扫描

示例：

```yaml
- name: Scan image
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
    format: table
    exit-code: "1"
    severity: HIGH,CRITICAL
```

注意：如果你的镜像没有用 `${{ github.sha }}` 作为 tag，就要改成真实 tag。

也可以在 build-push 前使用本地构建结果扫描，但需要根据 workflow 设计调整。

## 5. SBOM 是什么

SBOM 是 Software Bill of Materials，软件物料清单。

它记录镜像或应用包含了哪些组件和依赖。

常见格式：

- CycloneDX。
- SPDX。

生成 SBOM 的价值：

- 审计依赖。
- 漏洞响应。
- 供应链安全。
- 合规要求。

## 6. 用 Trivy 生成 SBOM

```bash
trivy image --format cyclonedx --output sbom.cdx.json go-cicd-lab:local
```

上传 artifact：

```yaml
- name: Generate SBOM
  run: trivy image --format cyclonedx --output sbom.cdx.json go-cicd-lab:local

- name: Upload SBOM
  uses: actions/upload-artifact@v4
  with:
    name: sbom
    path: sbom.cdx.json
    if-no-files-found: error
```

## 7. 基础加固清单

Go 服务镜像建议：

- 使用多阶段构建。
- 最终镜像不包含源码和编译器。
- 使用非 root 用户。
- 不把 `.env`、密钥、私钥 COPY 进镜像。
- 使用明确 tag。
- 记录 OCI labels。
- 定期更新基础镜像。
- 扫描 HIGH/CRITICAL 漏洞。
- 保留可回滚版本。

## 8. 选择基础镜像

常见选择：

| 镜像 | 优点 | 注意 |
| --- | --- | --- |
| `scratch` | 极小 | 需要自己处理 CA 证书等 |
| distroless | 小，攻击面低 | 没有 shell，调试方式不同 |
| alpine | 小，有包管理器 | musl/glibc 差异可能影响某些程序 |
| debian slim | 兼容性好 | 体积更大 |

Go 静态 HTTP 服务可以优先尝试 distroless。

## 9. 漏洞是否一定阻断发布

建议规则：

```text
CRITICAL 且可利用：阻断。
HIGH：通常阻断，除非有明确例外。
MEDIUM/LOW：记录并计划修复。
误报或不可达：记录说明和过期时间。
```

不要让安全例外永久存在。

## 10. 和第 8 阶段的关系

本阶段只做镜像层面的基础安全。

第 8 阶段会系统学习：

- CI/CD secret 安全。
- OIDC。
- 签名。
- provenance。
- SLSA。
- 更完整的供应链安全。

## 11. 小练习

完成：

1. 本地扫描镜像：

```bash
trivy image go-cicd-lab:local
```

2. 生成 SBOM：

```bash
trivy image --format cyclonedx --output sbom.cdx.json go-cicd-lab:local
```

3. 记录扫描结果：

```text
HIGH:
CRITICAL:
是否阻断:
修复方案:
```

## 12. 本节小结

你现在应该理解：

- 镜像扫描是发布前的重要门禁。
- SBOM 是软件依赖清单。
- 非 root、多阶段构建、不带密钥是基础加固。
- 安全扫描结果需要结合严重等级和可利用性判断。

