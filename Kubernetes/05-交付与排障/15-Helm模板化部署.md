# 15. Helm 模板化部署

本节目标：学完后你能用 Helm 把重复的 Kubernetes YAML 整理成可配置的部署包。

## 简短引入

一个后端服务通常有 Deployment、Service、ConfigMap、Ingress、Job。环境一多，复制 YAML 很容易改漏。Helm 可以理解为 Kubernetes YAML 的模板和打包工具。

## 一、为什么需要它

真实团队会遇到：

- dev、test、prod 镜像标签不同。
- 副本数不同。
- 域名不同。
- 配置不同。
- 多个服务结构类似。

Helm 的目标不是让部署变神秘，而是减少重复和手工改 YAML 的风险。

## 二、基本用法

创建 Chart：

```bash
helm create short-api
```

常见目录：

```text
short-api/
  Chart.yaml
  values.yaml
  templates/
```

简化后的 `values.yaml`：

```yaml
image:
  repository: short-api
  tag: "0.1.0"

replicaCount: 2

service:
  port: 8080

config:
  APP_ENV: dev
  LOG_LEVEL: debug
```

模板里使用：

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
replicas: {{ .Values.replicaCount }}
```

渲染检查：

```bash
helm template short-api ./short-api
```

安装或升级：

```bash
helm upgrade --install short-api ./short-api -n backend-dev --create-namespace
```

## 三、关键参数/语法/代码结构

- `Chart.yaml`：Chart 元信息。
- `values.yaml`：默认配置。
- `templates/`：模板文件。
- `helm template`：只渲染，不提交集群，适合检查。
- `helm upgrade --install`：不存在就安装，存在就升级。
- `-f values-prod.yaml`：指定环境配置。

```text
每次发布前先 helm template 或 helm diff，避免把模板错误直接打到集群。
```

## 四、真实后端场景示例

生产环境配置 `values-prod.yaml`：

```yaml
image:
  repository: registry.example.com/backend/short-api
  tag: "2026.07.06-abc123"

replicaCount: 4

config:
  APP_ENV: prod
  LOG_LEVEL: info

ingress:
  host: short.example.com
```

发布：

```bash
helm upgrade --install short-api ./short-api -n backend-prod -f values-prod.yaml
```

真实项目中，镜像标签通常由 CI/CD 注入，不靠开发手工修改。

## 五、注意点

- 不要把 Helm 模板写得过度复杂。
- Secret 不建议明文写进公开 values 文件。
- Chart 的默认值要适合开发环境，生产值单独管理。
- Helm 管的是 Kubernetes 资源，不替代应用配置校验和数据库迁移策略。

## 六、常见错误

- 错误：模板里到处写复杂判断。
  后期维护成本高，排障时很难看清最终 YAML。
- 错误：直接 `helm upgrade` 不看渲染结果。
  模板错误可能造成生产资源异常。
- 错误：把所有环境差异都塞一个 values 文件。
  环境边界不清晰，容易误发。

## 七、本节达标标准

- 能解释 Helm Chart 的基本结构。
- 能用 values 控制镜像、端口、副本数。
- 能用 `helm template` 检查渲染结果。
- 能用 `helm upgrade --install` 部署服务。

## 八、从渲染结果学习 Helm

Helm 最容易被模板语法吓住。先记住：集群最终接收的仍是普通 Kubernetes YAML。每次先渲染并检查关键字段：

```bash
helm lint ./short-api
helm template short-api ./short-api -f values.yaml
helm template short-api ./short-api -f values-prod.yaml --debug
```

重点确认资源名称、Namespace、镜像、标签、selector、端口、Ingress 域名和 Secret 引用。渲染正确后再安装：

```bash
helm upgrade --install short-api ./short-api \
  -n backend-dev --create-namespace --wait --timeout 5m
helm list -n backend-dev
helm status short-api -n backend-dev
```

`--wait` 会等待部分资源达到就绪状态，但不能代替业务冒烟测试。自测：values 提供数据，templates 描述数据如何组成最终 YAML；修改 values 后仍必须重新升级才会影响集群。
