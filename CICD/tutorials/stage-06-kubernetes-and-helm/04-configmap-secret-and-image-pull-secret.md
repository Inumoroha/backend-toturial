# 04：ConfigMap、Secret 与 imagePullSecrets

## 1. 本节目标

这一节把配置和密钥放进 Kubernetes。

你要学习：

- ConfigMap 存非敏感配置。
- Secret 存敏感配置。
- imagePullSecret 拉取私有镜像。
- 不把环境差异写进镜像。

## 2. ConfigMap

`deploy/k8s/base/configmap.yaml`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: go-cicd-lab-config
  namespace: go-cicd-lab
data:
  APP_ENV: "staging"
  HTTP_ADDR: ":8080"
  LOG_LEVEL: "info"
```

在 Deployment 中使用：

```yaml
envFrom:
  - configMapRef:
      name: go-cicd-lab-config
```

## 3. Secret

示例 Secret：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: go-cicd-lab-secret
  namespace: go-cicd-lab
type: Opaque
stringData:
  DATABASE_URL: "postgres://user:password@postgres:5432/go_cicd_lab?sslmode=disable"
  JWT_SECRET: "change-me"
```

在 Deployment 中使用：

```yaml
envFrom:
  - secretRef:
      name: go-cicd-lab-secret
```

注意：这个 YAML 里包含明文密钥，真实项目不要直接提交真实 Secret。

## 4. Secret 的现实问题

Kubernetes Secret 默认不是“神奇安全保险箱”。

你仍然要考虑：

- 谁能读取 Secret。
- etcd 是否加密。
- Secret 是否进入 Git。
- CI 日志是否打印。
- 本地 kubeconfig 权限。

学习阶段可以用 Secret 理解机制。

生产建议使用：

- 外部 secret manager。
- Sealed Secrets。
- External Secrets Operator。
- 云厂商 Secret Manager。

这些会在第 8 阶段深入。

## 5. Deployment 引用配置

片段：

```yaml
spec:
  template:
    spec:
      containers:
        - name: api
          image: go-cicd-lab:local
          envFrom:
            - configMapRef:
                name: go-cicd-lab-config
            - secretRef:
                name: go-cicd-lab-secret
```

配置变化后，Pod 不一定自动重启。

你可能需要：

```bash
kubectl -n go-cicd-lab rollout restart deployment/go-cicd-lab
```

Helm 中常用 checksum annotation 触发滚动更新，后面会讲。

## 6. imagePullSecret

如果镜像在私有 GHCR：

```bash
kubectl -n go-cicd-lab create secret docker-registry ghcr-pull \
  --docker-server=ghcr.io \
  --docker-username=<github-username> \
  --docker-password=<token> \
  --docker-email=<email>
```

Deployment 中引用：

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
        - name: ghcr-pull
```

如果镜像公开，通常不需要 imagePullSecret。

## 7. 不要提交真实 Secret

可以提交：

```text
secret.example.yaml
```

不要提交：

```text
secret.yaml
```

推荐 `.gitignore`：

```text
deploy/k8s/**/secret.yaml
deploy/k8s/**/*secret*.local.yaml
```

但更好的方式是让真实密钥由 CI/CD 或 secret manager 注入。

## 8. 小练习

1. 创建 ConfigMap。
2. 创建 Secret。
3. 修改 Deployment 使用 `envFrom`。
4. apply 后查看 Pod 环境变量：

```bash
kubectl -n go-cicd-lab exec deploy/go-cicd-lab -- env
```

如果镜像是 distroless，可能无法 exec shell，但 `env` 作为命令也可能不存在。此时更适合通过应用 `/config` 或日志确认非敏感配置，不要打印密钥。

## 9. 本节小结

你现在应该理解：

- ConfigMap 存非敏感配置。
- Secret 存敏感配置，但仍需权限和加密保护。
- 私有镜像需要 imagePullSecret。
- 配置和密钥不要写进镜像。
- 真实 Secret 不应该进 Git。

