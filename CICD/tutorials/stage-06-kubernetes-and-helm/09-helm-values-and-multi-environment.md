# 09：Helm Values 与多环境

## 1. 本节目标

这一节学习如何用 Helm 管理不同环境。

目标：

```text
同一个 Chart
不同 values
部署到 staging 和 production
```

## 2. 推荐 values 文件

```text
deploy/helm/go-cicd-lab/
  values.yaml
  values-staging.yaml
  values-production.yaml
```

`values.yaml` 放默认值。

环境文件只覆盖差异。

## 3. staging values

`values-staging.yaml`：

```yaml
replicaCount: 1

image:
  tag: "sha-example"
  pullPolicy: Always

config:
  APP_ENV: staging
  LOG_LEVEL: debug

ingress:
  enabled: true
  host: staging.go-cicd-lab.example.com

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 300m
    memory: 256Mi
```

## 4. production values

`values-production.yaml`：

```yaml
replicaCount: 3

image:
  pullPolicy: IfNotPresent

config:
  APP_ENV: production
  LOG_LEVEL: info

ingress:
  enabled: true
  host: go-cicd-lab.example.com

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 1Gi
```

## 5. 设置 image tag

不要把每次发布的 tag 写死在 values 文件里。

部署时传入：

```bash
helm upgrade --install go-cicd-lab deploy/helm/go-cicd-lab \
  --namespace go-cicd-lab \
  --create-namespace \
  -f deploy/helm/go-cicd-lab/values-staging.yaml \
  --set image.tag=sha-8f3a2c1
```

production：

```bash
helm upgrade --install go-cicd-lab deploy/helm/go-cicd-lab \
  --namespace go-cicd-lab \
  --create-namespace \
  -f deploy/helm/go-cicd-lab/values-production.yaml \
  --set image.tag=v0.1.0
```

## 6. Secret values 不要进 Git

不要在 values-production.yaml 写：

```yaml
secret:
  DATABASE_URL: postgres://prod-user:prod-password@...
```

推荐：

- Secret 由平台提前创建。
- Helm 只引用 Secret 名称。
- 或 CI 从 GitHub environment secrets 创建/更新 Secret。
- 或使用 External Secrets。

学习阶段可以用 `kubectl create secret` 手动创建。

## 7. imagePullSecrets

values：

```yaml
imagePullSecrets:
  - name: ghcr-pull
```

Deployment 模板：

```gotemplate
{{- with .Values.imagePullSecrets }}
imagePullSecrets:
  {{- toYaml . | nindent 8 }}
{{- end }}
```

注意缩进位置要在 `spec.template.spec` 下。

## 8. 检查不同环境渲染

```bash
helm template go-cicd-lab deploy/helm/go-cicd-lab \
  -f deploy/helm/go-cicd-lab/values-staging.yaml \
  --set image.tag=sha-test
```

```bash
helm template go-cicd-lab deploy/helm/go-cicd-lab \
  -f deploy/helm/go-cicd-lab/values-production.yaml \
  --set image.tag=v0.1.0
```

对比：

- replicas。
- APP_ENV。
- LOG_LEVEL。
- ingress host。
- resources。

## 9. 多环境原则

推荐：

- Chart 一份。
- values 多份。
- image tag 部署时传入。
- secret 不进 Git。
- production values 更保守。

不要为 staging 和 production 复制两套 Chart。

复制会导致漂移。

## 10. 小练习

1. 创建 `values-staging.yaml`。
2. 创建 `values-production.yaml`。
3. 分别 `helm template`。
4. 确认 production replicas 比 staging 多。
5. 确认 image tag 可以通过 `--set` 覆盖。

## 11. 本节小结

你现在应该理解：

- values 文件用于表达环境差异。
- image tag 应该在部署时传入。
- secret 不应该写进 values 文件。
- 同一 Chart 多环境 values 能减少配置漂移。

