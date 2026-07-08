# 08：Helm Chart 基础

## 1. 本节目标

手写 YAML 能帮助你理解 Kubernetes。

但真实项目会遇到：

- staging 和 production 配置不同。
- image tag 每次变。
- replicas 不同。
- ingress host 不同。
- resources 不同。

Helm 用模板和 values 管理这些差异。

## 2. Helm 是什么

Helm 是 Kubernetes 的包管理工具。

它提供：

- Chart。
- Values。
- Template。
- Release。
- Upgrade。
- Rollback。

一句话：

```text
Chart 是模板包，values 是输入参数，release 是安装到集群中的实例。
```

## 3. 创建 Chart

推荐目录：

```text
deploy/helm/go-cicd-lab/
```

可以手动创建，也可以：

```bash
helm create go-cicd-lab
```

初学时自动生成内容较多，建议你手动保留最小结构：

```text
deploy/helm/go-cicd-lab/
  Chart.yaml
  values.yaml
  templates/
    deployment.yaml
    service.yaml
    configmap.yaml
    secret.yaml
    ingress.yaml
    _helpers.tpl
```

## 4. Chart.yaml

```yaml
apiVersion: v2
name: go-cicd-lab
description: A Helm chart for go-cicd-lab
type: application
version: 0.1.0
appVersion: "0.1.0"
```

说明：

- `version`：Chart 版本。
- `appVersion`：应用版本。

不要把两者混淆。

## 5. values.yaml

```yaml
replicaCount: 2

image:
  repository: ghcr.io/your-name/go-cicd-lab
  tag: "sha-example"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

config:
  APP_ENV: staging
  HTTP_ADDR: ":8080"
  LOG_LEVEL: info

secret:
  create: false
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

## 6. _helpers.tpl

`templates/_helpers.tpl`：

```gotemplate
{{- define "go-cicd-lab.name" -}}
go-cicd-lab
{{- end -}}

{{- define "go-cicd-lab.fullname" -}}
{{ .Release.Name }}
{{- end -}}

{{- define "go-cicd-lab.labels" -}}
app.kubernetes.io/name: {{ include "go-cicd-lab.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}
{{- end -}}
```

## 7. Deployment 模板

`templates/deployment.yaml`：

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
            {{- if .Values.secret.name }}
            - secretRef:
                name: {{ .Values.secret.name }}
            {{- end }}
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

## 8. 渲染模板

```bash
helm template go-cicd-lab deploy/helm/go-cicd-lab
```

带 namespace：

```bash
helm template go-cicd-lab deploy/helm/go-cicd-lab --namespace go-cicd-lab
```

## 9. 常见 Helm 模板函数

- `include`
- `nindent`
- `toYaml`
- `default`
- `quote`
- `required`

初学不要过度模板化。

先让 Chart 清晰可读。

## 10. 小练习

1. 创建最小 Helm Chart。
2. 添加 Deployment 模板。
3. 添加 Service 模板。
4. 执行：

```bash
helm template go-cicd-lab deploy/helm/go-cicd-lab
helm lint deploy/helm/go-cicd-lab
```

5. 修复模板错误。

## 11. 本节小结

你现在应该理解：

- Chart 是 Helm 包。
- values 控制模板输出。
- release 是安装实例。
- `helm template` 是学习和排查 Helm 的第一工具。

