# 12：实战，部署 go-cicd-lab 到 Kubernetes

## 1. 本节目标

这一节把第 6 阶段落到 `go-cicd-lab`。

最终产出：

```text
deploy/k8s/base/
deploy/helm/go-cicd-lab/
.github/workflows/k8s.yml
docs/kubernetes.md
```

## 2. 创建基础 manifests

目录：

```text
deploy/k8s/base/
```

文件：

```text
namespace.yaml
configmap.yaml
deployment.yaml
service.yaml
```

先用这些手写 YAML 理解资源，再用 Helm 管理。

## 3. 创建 Helm Chart

目录：

```text
deploy/helm/go-cicd-lab/
  Chart.yaml
  values.yaml
  values-staging.yaml
  values-production.yaml
  templates/
    _helpers.tpl
    configmap.yaml
    deployment.yaml
    service.yaml
    ingress.yaml
```

## 4. Chart.yaml

```yaml
apiVersion: v2
name: go-cicd-lab
description: Go CI/CD lab service
type: application
version: 0.1.0
appVersion: "0.1.0"
```

## 5. values.yaml

```yaml
replicaCount: 1

image:
  repository: ghcr.io/your-name/go-cicd-lab
  tag: "sha-example"
  pullPolicy: IfNotPresent

imagePullSecrets: []

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

config:
  APP_ENV: staging
  HTTP_ADDR: ":8080"
  LOG_LEVEL: info

secret:
  name: go-cicd-lab-secret

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

ingress:
  enabled: false
  className: nginx
  host: go-cicd-lab.local
```

## 6. configmap.yaml

```gotemplate
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "go-cicd-lab.fullname" . }}
  labels:
    {{- include "go-cicd-lab.labels" . | nindent 4 }}
data:
  {{- range $key, $value := .Values.config }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
```

## 7. service.yaml

```gotemplate
apiVersion: v1
kind: Service
metadata:
  name: {{ include "go-cicd-lab.fullname" . }}
  labels:
    {{- include "go-cicd-lab.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  selector:
    app.kubernetes.io/name: {{ include "go-cicd-lab.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
  ports:
    - name: http
      port: {{ .Values.service.port }}
      targetPort: http
```

## 8. deployment.yaml

```gotemplate
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "go-cicd-lab.fullname" . }}
  labels:
    {{- include "go-cicd-lab.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "go-cicd-lab.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ include "go-cicd-lab.name" . }}
        app.kubernetes.io/instance: {{ .Release.Name }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: api
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
          envFrom:
            - configMapRef:
                name: {{ include "go-cicd-lab.fullname" . }}
            - secretRef:
                name: {{ .Values.secret.name }}
          startupProbe:
            httpGet:
              path: /healthz
              port: http
            failureThreshold: 30
            periodSeconds: 2
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            periodSeconds: 5
            timeoutSeconds: 2
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            periodSeconds: 10
            timeoutSeconds: 2
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

## 9. 本地安装

先创建 Secret：

```bash
kubectl create namespace go-cicd-lab --dry-run=client -o yaml | kubectl apply -f -
kubectl -n go-cicd-lab create secret generic go-cicd-lab-secret \
  --from-literal=DATABASE_URL='postgres://user:password@postgres:5432/go_cicd_lab?sslmode=disable' \
  --from-literal=JWT_SECRET='change-me'
```

安装：

```bash
helm upgrade --install go-cicd-lab deploy/helm/go-cicd-lab \
  --namespace go-cicd-lab \
  --create-namespace \
  -f deploy/helm/go-cicd-lab/values-staging.yaml \
  --set image.repository=go-cicd-lab \
  --set image.tag=local \
  --wait \
  --timeout 3m
```

如果用 kind 本地镜像，记得：

```bash
kind load docker-image go-cicd-lab:local --name go-cicd-lab
```

## 10. 访问服务

```bash
kubectl -n go-cicd-lab port-forward service/go-cicd-lab 8080:80
```

另一个终端：

```bash
curl -f http://127.0.0.1:8080/healthz
curl -f http://127.0.0.1:8080/readyz
```

## 11. CI workflow

创建：

```text
.github/workflows/k8s.yml
```

内容：

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
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: azure/setup-helm@v4
      - run: helm lint deploy/helm/go-cicd-lab
      - run: |
          helm template go-cicd-lab deploy/helm/go-cicd-lab \
            -f deploy/helm/go-cicd-lab/values-staging.yaml \
            --set image.tag=sha-test
      - run: |
          helm template go-cicd-lab deploy/helm/go-cicd-lab \
            -f deploy/helm/go-cicd-lab/values-production.yaml \
            --set image.tag=v0.1.0
```

## 12. 文档

创建：

```text
docs/kubernetes.md
```

内容模板：

````markdown
# Kubernetes 部署说明

## 本地集群

```bash
kind create cluster --name go-cicd-lab
kubectl create namespace go-cicd-lab
```

## 本地镜像

```bash
docker build -t go-cicd-lab:local .
kind load docker-image go-cicd-lab:local --name go-cicd-lab
```

## 安装

```bash
helm upgrade --install go-cicd-lab deploy/helm/go-cicd-lab \
  --namespace go-cicd-lab \
  --create-namespace \
  -f deploy/helm/go-cicd-lab/values-staging.yaml \
  --set image.repository=go-cicd-lab \
  --set image.tag=local \
  --wait
```

## 验证

```bash
kubectl -n go-cicd-lab get pods
kubectl -n go-cicd-lab rollout status deployment/go-cicd-lab
kubectl -n go-cicd-lab port-forward service/go-cicd-lab 8080:80
curl -f http://127.0.0.1:8080/healthz
```

## 回滚

```bash
helm history go-cicd-lab -n go-cicd-lab
helm rollback go-cicd-lab 1 -n go-cicd-lab
```
````

## 13. 验收

你完成后应该能做到：

- kind 中运行 Go 服务。
- Helm Chart 通过 lint 和 template。
- Helm install/upgrade 成功。
- probes 正常。
- port-forward 可访问。
- Helm rollback 可用。
- CI 能校验 Chart。

## 14. 本节小结

你已经把 `go-cicd-lab` 从 Compose 部署推进到 Kubernetes + Helm 部署。

这为第 7 阶段 GitOps 打好了基础。

