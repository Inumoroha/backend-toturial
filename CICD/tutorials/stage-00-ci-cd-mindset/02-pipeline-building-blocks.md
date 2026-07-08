# 02：流水线的组成部分

## 1. 什么是 Pipeline

Pipeline 可以理解为一条自动化生产线。

在工厂里，产品会经过检查、组装、包装、出库。在软件里，一次代码变更会经过测试、构建、扫描、发布、验证。

一个典型后端 CI/CD Pipeline：

```text
trigger
-> checkout
-> setup runtime
-> install dependencies
-> lint
-> test
-> build
-> package
-> scan
-> publish artifact
-> deploy
-> verify
```

不同平台的语法不同，但核心组件大多类似。

## 2. Trigger：触发器

Trigger 决定流水线什么时候运行。

常见触发方式：

- push：有人推送代码。
- pull request：有人提交合并请求。
- tag：有人推送版本标签，例如 `v1.0.0`。
- schedule：定时运行，例如每天凌晨扫描依赖漏洞。
- manual：手动点击运行。

Go 后端常见设计：

| 触发条件 | 适合做什么 |
| --- | --- |
| pull request | 测试、lint、构建检查 |
| push main | 构建镜像、部署 staging |
| tag v* | 发布正式版本 |
| manual | 手动部署、回滚、补跑任务 |
| schedule | 安全扫描、依赖检查 |

## 3. Job：任务

Job 是流水线中的一个工作单元。

例如：

```text
ci pipeline
  job: lint
  job: test
  job: build
  job: scan
```

Job 可以串行，也可以并行。

并行示例：

```text
          -> lint
trigger   -> test
          -> build
```

串行示例：

```text
test -> build -> deploy
```

通常，部署 job 必须等测试和构建成功之后再运行。

## 4. Step：步骤

Step 是 job 里面的具体命令或动作。

例如一个 `test` job 可能包含：

```text
step 1: 拉取代码
step 2: 安装 Go
step 3: 下载依赖
step 4: 执行 go test ./...
step 5: 上传覆盖率报告
```

你可以把关系理解成：

```text
pipeline 包含多个 job
job 包含多个 step
step 执行具体命令
```

## 5. Runner/Agent：执行机器

Runner 是真正执行流水线任务的机器或容器。

它可能是：

- CI 平台托管的虚拟机。
- 公司自己维护的服务器。
- Kubernetes 里的临时 Pod。
- Docker 容器。

你需要关心 Runner 上有什么：

- 操作系统是什么。
- 有没有 Go。
- 有没有 Docker。
- 有没有访问内网的权限。
- 有没有足够 CPU、内存、磁盘。
- 有没有正确的网络访问权限。

很多“代码没问题但 CI 失败”的原因，都来自 Runner 环境差异。

## 6. Cache：缓存

Cache 用来加速流水线。

例如 Go 项目可以缓存：

- module download cache。
- build cache。
- Docker layer cache。

没有缓存时：

```text
每次 CI 都重新下载所有依赖 -> 慢
```

有缓存时：

```text
依赖没有变化 -> 复用缓存 -> 快
```

注意：cache 是性能优化，不应该影响最终正确性。也就是说，即使缓存失效，流水线应该仍然能跑通，只是变慢。

## 7. Artifact：制品

Artifact 是流水线生成并需要保存或传递的结果。

常见 artifact：

- Go 二进制文件。
- 测试报告。
- 覆盖率报告。
- Docker 镜像。
- Helm Chart。
- SBOM 文件。

cache 和 artifact 的区别：

| 项目 | 用途 | 是否代表最终结果 |
| --- | --- | --- |
| cache | 加速下一次运行 | 不是 |
| artifact | 保存或交付本次构建结果 | 是 |

一句话记忆：

```text
cache 是为了跑得快。
artifact 是为了交付和追溯。
```

## 8. Secret：密钥

Secret 是流水线中需要保护的敏感信息。

例如：

- 镜像仓库 token。
- 云厂商访问密钥。
- SSH 私钥。
- 数据库密码。
- Kubernetes kubeconfig。
- 第三方服务 API key。

基本原则：

- 不写进代码仓库。
- 不打印到日志。
- 不复制进 Docker 镜像。
- 不给所有 job 默认开放。
- 不给 PR from fork 随便使用生产密钥。

## 9. Environment：环境

Environment 表示代码运行的目标环境。

常见环境：

- local：开发者本地。
- dev：开发环境。
- test：测试环境。
- staging：预发布环境，尽量模拟生产。
- production：生产环境。

CI/CD 经常围绕环境设计规则：

```text
main 自动部署 staging
production 需要审批
production 只能使用版本 tag
production 密钥只给生产部署 job
```

## 10. Gate：门禁

Gate 是进入下一阶段前必须满足的条件。

常见门禁：

- 测试通过。
- 代码评审通过。
- 覆盖率不低于阈值。
- 没有高危漏洞。
- 镜像扫描通过。
- 人工审批通过。
- 冒烟测试通过。

门禁的本质是：把风险挡在更早的位置。

## 11. 小练习

请把下面这些词分类到 trigger、job、step、cache、artifact、secret、environment、gate 中：

- `go test ./...`
- `main branch push`
- `staging`
- `coverage.out`
- `Docker image`
- `KUBE_CONFIG`
- `manual approval`
- `Go module cache`
- `build job`
- `pull request`

建议你自己先分类，再看答案。

参考答案：

| 内容 | 分类 |
| --- | --- |
| `go test ./...` | step |
| `main branch push` | trigger |
| `staging` | environment |
| `coverage.out` | artifact |
| `Docker image` | artifact |
| `KUBE_CONFIG` | secret |
| `manual approval` | gate |
| `Go module cache` | cache |
| `build job` | job |
| `pull request` | trigger |

## 12. 本节小结

你现在应该能理解：

- Pipeline 是自动化流程。
- Trigger 决定何时运行。
- Job 是任务单元。
- Step 是具体命令。
- Runner 是执行环境。
- Cache 用来加速。
- Artifact 用来交付和追溯。
- Secret 必须保护。
- Environment 决定部署目标。
- Gate 控制风险进入下一阶段。

