# 11：CI 校验与 Helm 部署

## 1. 本节目标

第 3 阶段你已经会写 CI。

第 6 阶段要把 Kubernetes/Helm 加进 CI/CD：

```text
PR:
  helm lint
  helm template
  optional kubeconform

main:
  deploy staging with helm

tag v*:
  deploy production with approval
```

## 2. PR 阶段只校验，不部署

PR 推荐：

```text
helm lint
helm template
kubectl apply --dry-run=client
```

更严格可用 kubeconform/kubeval 校验 Kubernetes schema。

初学先用 Helm 自带命令。

## 3. GitHub Actions 校验示例

`.github/workflows/k8s.yml`：

```yaml
name: k8s

on:
  pull_request:
    branches: [main]
    paths:
      - "deploy/helm/**"
      - ".github/workflows/k8s.yml"
  push:
    branches: [main]
    paths:
      - "deploy/helm/**"
      - ".github/workflows/k8s.yml"

permissions:
  contents: read

jobs:
  validate:
    name: validate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - name: Setup Helm
        uses: azure/setup-helm@v4

      - name: Helm lint
        run: helm lint deploy/helm/go-cicd-lab

      - name: Helm template staging
        run: |
          helm template go-cicd-lab deploy/helm/go-cicd-lab \
            -f deploy/helm/go-cicd-lab/values-staging.yaml \
            --set image.tag=sha-test

      - name: Helm template production
        run: |
          helm template go-cicd-lab deploy/helm/go-cicd-lab \
            -f deploy/helm/go-cicd-lab/values-production.yaml \
            --set image.tag=v0.1.0
```

## 4. 部署到 Kubernetes 需要什么凭证

CI 部署到集群需要访问 Kubernetes API。

常见方式：

- kubeconfig 存在 environment secret。
- 云厂商 OIDC 获取短期凭证。
- self-hosted runner 在集群网络内。
- GitOps 控制器拉取配置，CI 不直接访问集群。

本阶段先用 kubeconfig 学习。

第 7 阶段会学习 GitOps，减少 CI 直接持有集群部署权限。

## 5. kubeconfig Secret

在 GitHub environment 中保存：

```text
KUBE_CONFIG
```

部署 job 中写入：

```yaml
- name: Configure kubeconfig
  run: |
    mkdir -p ~/.kube
    printf '%s' "${{ secrets.KUBE_CONFIG }}" > ~/.kube/config
    chmod 600 ~/.kube/config
```

注意：

- production kubeconfig 不给 PR。
- kubeconfig 权限要最小。
- 最好使用 namespace-scoped 权限。

## 6. staging 部署示例

```yaml
  deploy-staging:
    name: deploy-staging
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: staging
    steps:
      - uses: actions/checkout@v6

      - name: Setup Helm
        uses: azure/setup-helm@v4

      - name: Configure kubeconfig
        run: |
          mkdir -p ~/.kube
          printf '%s' "${{ secrets.KUBE_CONFIG }}" > ~/.kube/config
          chmod 600 ~/.kube/config

      - name: Set image tag
        run: echo "IMAGE_TAG=sha-${GITHUB_SHA::7}" >> "$GITHUB_ENV"

      - name: Deploy with Helm
        run: |
          helm upgrade --install go-cicd-lab deploy/helm/go-cicd-lab \
            --namespace go-cicd-lab-staging \
            --create-namespace \
            -f deploy/helm/go-cicd-lab/values-staging.yaml \
            --set image.tag="$IMAGE_TAG" \
            --wait \
            --timeout 3m \
            --atomic
```

## 7. production 部署示例

```yaml
  deploy-production:
    name: deploy-production
    if: github.event_name == 'push' && startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    environment:
      name: production
    steps:
      - uses: actions/checkout@v6
      - uses: azure/setup-helm@v4
      - name: Configure kubeconfig
        run: |
          mkdir -p ~/.kube
          printf '%s' "${{ secrets.KUBE_CONFIG }}" > ~/.kube/config
          chmod 600 ~/.kube/config
      - name: Deploy with Helm
        run: |
          helm upgrade --install go-cicd-lab deploy/helm/go-cicd-lab \
            --namespace go-cicd-lab-production \
            --create-namespace \
            -f deploy/helm/go-cicd-lab/values-production.yaml \
            --set image.tag="${GITHUB_REF_NAME}" \
            --wait \
            --timeout 5m \
            --atomic
```

production environment 应配置 required reviewers。

## 8. 部署后验证

Helm `--wait` 只等待 Kubernetes 资源 ready。

你仍然可以增加冒烟测试：

```yaml
- name: Smoke test
  run: |
    kubectl -n go-cicd-lab-staging rollout status deployment/go-cicd-lab
    kubectl -n go-cicd-lab-staging port-forward service/go-cicd-lab 8080:80 &
    sleep 5
    curl -fsS http://127.0.0.1:8080/healthz
    curl -fsS http://127.0.0.1:8080/readyz
```

真实环境更常用外部 URL 测试。

## 9. 小练习

1. 为 Helm Chart 添加 validate workflow。
2. PR 中故意写错模板，观察失败。
3. 在 staging environment 配置 kubeconfig。
4. main push 部署 staging。
5. tag 触发 production，观察审批。

## 10. 本节小结

你现在应该理解：

- PR 阶段校验 Chart，不部署。
- main 可以自动部署 staging。
- tag 可以审批后部署 production。
- kubeconfig 是敏感凭证，要放 environment secret。
- `--wait --atomic` 能提高 Helm 部署安全性，但不能替代业务冒烟测试。

