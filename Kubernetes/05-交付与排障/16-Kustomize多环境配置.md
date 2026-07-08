# 16. Kustomize 多环境配置

本节目标：学完后你能用 Kustomize 在不写模板语法的情况下管理 dev、prod 环境差异。

## 简短引入

Helm 是模板化思路，Kustomize 是补丁叠加思路。它适合你已经有一组基础 YAML，然后不同环境只改少量字段的场景。

## 一、为什么需要它

真实项目中 dev 和 prod 资源结构大体一样，但有这些差异：

- namespace 不同。
- 副本数不同。
- 镜像标签不同。
- Ingress 域名不同。
- 日志级别不同。

如果复制两份完整 YAML，后续改 Service 端口时很容易只改一边。

## 二、基本用法

目录结构：

```text
k8s/
  base/
    deployment.yaml
    service.yaml
    kustomization.yaml
  overlays/
    dev/
      kustomization.yaml
      patch-deployment.yaml
    prod/
      kustomization.yaml
      patch-deployment.yaml
```

`base/kustomization.yaml`：

```yaml
resources:
  - deployment.yaml
  - service.yaml
```

`overlays/dev/kustomization.yaml`：

```yaml
namespace: backend-dev
resources:
  - ../../base
patches:
  - path: patch-deployment.yaml
images:
  - name: short-api
    newTag: "0.1.0-dev"
```

`overlays/dev/patch-deployment.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: short-api
spec:
  replicas: 1
```

应用：

```bash
kubectl apply -k k8s/overlays/dev
```

## 三、关键参数/语法/代码结构

- `resources`：引用基础资源。
- `namespace`：给资源统一设置 namespace。
- `patches`：对基础资源打补丁。
- `images`：替换镜像名称或标签。
- `kubectl apply -k`：直接应用 Kustomize 目录。

查看渲染结果：

```bash
kubectl kustomize k8s/overlays/dev
```

## 四、真实后端场景示例

短链接服务生产环境副本更多，日志级别更保守：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: short-api
spec:
  replicas: 4
  template:
    spec:
      containers:
        - name: short-api
          env:
            - name: LOG_LEVEL
              value: "info"
```

真实团队里，Kustomize 很适合“平台统一提供 base，业务项目维护 overlay”的协作方式。

## 五、注意点

- base 里不要写环境强相关配置。
- overlay 只放环境差异，别复制整份资源。
- patch 要尽量小，越大越容易和 base 漂移。
- Kustomize 不负责发布历史管理，回滚仍要结合 Git、CI/CD 或 Deployment rollout。

```text
Kustomize 的核心习惯是：base 表达共同结构，overlay 表达环境差异。
```

## 六、常见错误

- 错误：每个 overlay 复制完整 Deployment。
  这会失去 Kustomize 的意义。
- 错误：base 写死生产域名。
  dev 环境复用时容易误连生产。
- 错误：不看最终渲染结果。
  补丁可能没打上，或者打到了错误字段。

## 七、本节达标标准

- 能搭建 base/overlay 目录。
- 能用 Kustomize 修改副本数和镜像标签。
- 能查看最终渲染 YAML。
- 能区分 Helm 和 Kustomize 的适用场景。

