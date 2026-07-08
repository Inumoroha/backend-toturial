# 04：镜像 Tag、Label、Digest 与版本策略

## 1. 本节目标

镜像构建出来之后，必须能追踪。

你要能回答：

```text
这个镜像来自哪个 commit？
这是 main 分支构建还是正式 release？
生产现在运行的是哪个 digest？
失败后要回滚到哪个 tag？
```

## 2. Tag 是什么

镜像 tag 是镜像的人类可读版本名。

示例：

```text
go-cicd-lab:local
ghcr.io/your-name/go-cicd-lab:main
ghcr.io/your-name/go-cicd-lab:main-8f3a2c1
ghcr.io/your-name/go-cicd-lab:v0.1.0
ghcr.io/your-name/go-cicd-lab:sha-8f3a2c1
```

tag 可以移动。

例如 `main` 每次 main 分支构建都可能指向新镜像。

## 3. Digest 是什么

digest 是镜像内容的不可变标识。

示例：

```text
ghcr.io/your-name/go-cicd-lab@sha256:abc123...
```

tag 像名字，digest 像内容指纹。

生产部署最严格的方式是使用 digest，因为它不可变。

初学阶段可以先用明确版本 tag，后面逐步理解 digest 部署。

## 4. 不要只依赖 latest

不要把生产唯一版本写成：

```text
go-cicd-lab:latest
```

问题：

- 不知道它来自哪个 commit。
- 可能被新构建覆盖。
- 回滚不清晰。
- 审计困难。

`latest` 可以作为辅助标签，但不要作为生产唯一依据。

## 5. 推荐标签策略

对学习项目推荐：

| 场景 | Tag 示例 | 用途 |
| --- | --- | --- |
| 本地构建 | `go-cicd-lab:local` | 本地调试 |
| PR 构建 | 不推送，或 `pr-12` | 验证构建 |
| main 构建 | `main-8f3a2c1` | staging 候选 |
| commit 精确标签 | `sha-8f3a2c1` | 精确追溯 |
| release | `v0.1.0` | 正式版本 |
| latest | `latest` | 辅助，不作生产唯一依据 |

## 6. 版本和 commit 同时打 tag

正式 release 建议同时生成：

```text
ghcr.io/your-name/go-cicd-lab:v0.1.0
ghcr.io/your-name/go-cicd-lab:sha-8f3a2c1
```

这样：

- 人看到 `v0.1.0` 容易理解。
- 系统用 `sha-8f3a2c1` 容易追踪。

## 7. Label 是什么

镜像 label 是附加 metadata。

常见 OCI labels：

```text
org.opencontainers.image.title
org.opencontainers.image.description
org.opencontainers.image.source
org.opencontainers.image.revision
org.opencontainers.image.version
org.opencontainers.image.created
```

Dockerfile 中可以写：

```dockerfile
LABEL org.opencontainers.image.title="go-cicd-lab"
LABEL org.opencontainers.image.description="Go CI/CD learning service"
```

更推荐在 CI 中用 `docker/metadata-action` 自动生成 labels。

## 8. 查看 labels

```bash
docker inspect go-cicd-lab:local
```

只看 labels：

```bash
docker inspect --format='{{json .Config.Labels}}' go-cicd-lab:local
```

PowerShell 也可以直接先用完整 `docker inspect`。

## 9. GitHub Actions 中自动生成 tag

后面会使用：

```yaml
- name: Docker metadata
  id: meta
  uses: docker/metadata-action@v5
  with:
    images: ghcr.io/your-name/go-cicd-lab
    tags: |
      type=ref,event=branch
      type=sha,prefix=sha-
      type=semver,pattern={{version}}
```

含义：

- branch 构建生成分支 tag。
- 每次构建生成 sha tag。
- Git tag `v0.1.0` 生成版本 tag。

## 10. 小练习

本地构建两个 tag：

```bash
docker build -t go-cicd-lab:local -t go-cicd-lab:sha-$(git rev-parse --short HEAD) .
```

查看：

```bash
docker images go-cicd-lab
docker inspect go-cicd-lab:local
```

思考：

1. 哪个 tag 适合本地调试？
2. 哪个 tag 适合 staging？
3. 哪个 tag 适合 production？
4. 哪个信息能帮你回滚？

## 11. 本节小结

你现在应该理解：

- tag 是可读版本名，但可能移动。
- digest 是内容指纹，更适合严格不可变部署。
- 生产不要只依赖 `latest`。
- 镜像应该同时具备人类可读版本和 commit 追踪信息。
- labels 能保存 source、revision、created 等 metadata。

