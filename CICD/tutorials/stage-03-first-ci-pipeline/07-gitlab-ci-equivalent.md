# 07：GitLab CI/CD 对照

## 1. 本节目标

主线教程使用 GitHub Actions，但很多公司使用 GitLab CI/CD。

这一节用 GitLab 写一条等价的 Go CI。

你要理解概念映射：

| GitHub Actions | GitLab CI/CD |
| --- | --- |
| workflow | pipeline |
| job | job |
| step | script 中的命令 |
| runner | runner |
| `on` | `rules` / pipeline source |
| artifact | artifacts |
| cache | cache |
| secret | CI/CD variables |

## 2. 配置文件位置

GitLab CI 配置文件在仓库根目录：

```text
.gitlab-ci.yml
```

GitLab 会读取这个文件创建 pipeline。

## 3. 最小 Go CI

```yaml
stages:
  - test

variables:
  GO_VERSION: "1.25"

go-test:
  stage: test
  image: golang:${GO_VERSION}
  script:
    - go version
    - go mod download
    - go vet ./...
    - go test ./...
    - go test -race ./...
    - go build ./...
```

这个 job 使用官方 Go 镜像执行命令。

## 4. 只在 MR 和 main 上跑

可以使用 `rules`：

```yaml
go-test:
  stage: test
  image: golang:${GO_VERSION}
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
  script:
    - go mod download
    - go test ./...
    - go build ./...
```

含义：

- Merge Request 运行。
- main 分支运行。

## 5. Cache

GitLab cache 示例：

```yaml
variables:
  GOPATH: "$CI_PROJECT_DIR/.go"

cache:
  key:
    files:
      - go.sum
  paths:
    - .go/pkg/mod/
    - .cache/go-build/
```

结合脚本：

```yaml
before_script:
  - mkdir -p .go .cache/go-build
  - go env -w GOPATH="$GOPATH"
  - go env -w GOCACHE="$CI_PROJECT_DIR/.cache/go-build"
```

说明：

- cache key 和 `go.sum` 绑定。
- 依赖变化时缓存自动变化。

## 6. Artifact

保存覆盖率：

```yaml
coverage:
  stage: test
  image: golang:${GO_VERSION}
  script:
    - go test -coverprofile=coverage.out ./...
    - go tool cover -func=coverage.out
  artifacts:
    when: always
    paths:
      - coverage.out
    expire_in: 7 days
```

artifact 是本次 job 的输出，可以下载查看。

## 7. Lint 和 vuln

可以拆成多个 job：

```yaml
stages:
  - test
  - lint
  - security

variables:
  GO_VERSION: "1.25"

test:
  stage: test
  image: golang:${GO_VERSION}
  script:
    - go mod download
    - go vet ./...
    - go test ./...
    - go test -race ./...
    - go build ./...

lint:
  stage: lint
  image: golangci/golangci-lint:v2.12.0
  script:
    - golangci-lint run

vuln:
  stage: security
  image: golang:${GO_VERSION}
  script:
    - go install golang.org/x/vuln/cmd/govulncheck@latest
    - govulncheck ./...
```

注意：镜像版本要根据你项目实际情况和官方镜像更新调整。

## 8. GitLab CI/CD Variables

GitLab 中 secret 通常放在：

```text
Settings -> CI/CD -> Variables
```

第 3 阶段不建议 PR/MR CI 使用生产 secret。

同样遵守：

- 最小权限。
- 按环境隔离。
- 不打印变量。
- 不让不受信任代码拿生产凭证。

## 9. GitLab 和 GitHub 的选择

学习时优先选一个平台深入。

建议：

- 个人项目或开源项目：GitHub Actions。
- 公司使用 GitLab：优先 GitLab CI。

不要同时在两个平台上写复杂流水线。先掌握模型，再迁移语法。

## 10. 小练习

如果你有 GitLab 仓库：

1. 创建 `.gitlab-ci.yml`。
2. 添加 `test` job。
3. 提交 MR。
4. 查看 pipeline。
5. 增加 cache。
6. 上传 `coverage.out` artifact。

## 11. 本节小结

你现在应该理解：

- GitLab CI 的配置文件是 `.gitlab-ci.yml`。
- GitLab 使用 stages 和 jobs 组织 pipeline。
- GitLab 的 `script` 对应 GitHub Actions 的多个 `run` step。
- cache、artifact、variables 的概念和 GitHub Actions 类似。
- CI/CD 模型比平台语法更重要。

