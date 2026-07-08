# 22. CI/CD 与镜像发布流水线

本节目标：学完后你能设计一条从 Go 代码提交到 Kubernetes 发布的保守 CI/CD 流水线。

## 简短引入

手工在本机构建镜像、手工 `kubectl apply` 适合学习，不适合团队长期协作。真实项目需要 CI/CD 把测试、构建、推送、部署、观察和回滚串起来。

## 一、为什么需要它

团队发布最怕这些问题：

- 不知道生产镜像来自哪次提交。
- 跳过测试直接发布。
- 多个人同时手工改集群。
- Secret 泄露在流水线日志里。
- 发布失败后不知道如何回滚。

CI/CD 的核心价值是让发布过程可重复、可审计、可回滚。

```text
流水线不是为了炫技，而是为了减少手工发布的不确定性。
```

## 二、基本用法

一条保守流水线可以这样设计：

```text
代码提交 -> 单元测试 -> 构建镜像 -> 镜像扫描 -> 推送镜像仓库 -> 渲染 YAML -> 部署测试环境 -> 运行冒烟测试 -> 人工审批 -> 部署生产 -> 观察指标 -> 必要时回滚
```

GitHub Actions 示例片段：

```yaml
name: short-api-ci

on:
  push:
    branches: [main]

jobs:
  test-and-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.22"
      - name: Test
        run: go test ./...
      - name: Build image
        run: docker build -t registry.example.com/backend/short-api:${{ github.sha }} .
```

部署命令示例：

```bash
helm upgrade --install short-api ./deploy/short-api \
  -n backend-prod \
  -f ./deploy/short-api/values-prod.yaml \
  --set image.tag=$GIT_COMMIT
```

## 三、关键参数/语法/代码结构

- 镜像标签：建议使用 Git commit SHA、构建号或语义版本。
- 测试：至少包含单元测试和关键接口冒烟测试。
- 部署身份：CI/CD 账号只给目标 namespace 的必要权限。
- 环境变量：Secret 走流水线密钥管理，不打印到日志。
- 渲染检查：Helm 用 `helm template`，Kustomize 用 `kubectl kustomize`。

上线前可以让流水线执行：

```bash
helm template short-api ./deploy/short-api -f ./deploy/short-api/values-prod.yaml
kubectl diff -n backend-prod -f rendered.yaml
```

`kubectl diff` 能让你看到即将变更的资源。生产中是否允许流水线执行 diff，要看权限策略。

## 四、真实后端场景示例

短链接服务一次生产发布：

1. 合并代码到 `main`。
2. CI 执行 `go test ./...`。
3. 构建镜像 `short-api:<commit-sha>`。
4. 推送到镜像仓库。
5. 执行数据库兼容迁移 Job。
6. Helm 更新 Deployment 镜像标签。
7. 等待 `kubectl rollout status` 成功。
8. 请求 `/healthz` 和核心创建短链接口做冒烟测试。
9. 观察 5xx、P95 延迟和数据库连接池。
10. 异常时先判断是否需要回滚代码，数据库回滚单独评估。

```text
回滚按钮只能解决一部分问题。凡是涉及数据库结构和数据修改，都要提前设计兼容发布和恢复方案。
```

## 五、注意点

- 生产发布不应该依赖开发者本机环境。
- CI/CD 的 kubeconfig 和 registry token 要定期轮换。
- 不要在流水线日志输出 Secret。
- 镜像构建要可复现，依赖版本要尽量固定。
- 数据库迁移失败时，流水线应该停止后续部署。
- 发布完成不等于上线成功，还要观察指标和业务结果。

## 六、常见错误

- 错误：镜像标签一直用 `latest`。
  回滚和审计都会变得困难。
- 错误：CI/CD 账号拥有集群管理员权限。
  流水线泄露或脚本错误时影响范围过大。
- 错误：测试环境没跑通就允许发生产。
  这会把低级配置错误直接带到线上。
- 错误：迁移失败还继续发布应用。
  新代码可能依赖不存在的表或字段。

## 七、本节达标标准

- 能画出一条后端服务 CI/CD 发布流程。
- 能解释镜像标签为什么要可追溯。
- 能说明 CI/CD 账号为什么要最小权限。
- 能把数据库迁移、发布、冒烟测试、观察和回滚串成闭环。
