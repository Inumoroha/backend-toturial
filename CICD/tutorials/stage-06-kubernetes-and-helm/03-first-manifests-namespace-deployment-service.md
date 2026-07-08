# 03：第一个 Manifests：Namespace、Deployment、Service

## 1. 本节目标

这一节手写第一组 Kubernetes YAML：

```text
Namespace
Deployment
Service
```

先不用 Helm，先理解原始资源。

## 2. 推荐目录

```text
deploy/k8s/base/
  namespace.yaml
  deployment.yaml
  service.yaml
```

## 3. Namespace

`deploy/k8s/base/namespace.yaml`：

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: go-cicd-lab
```

应用：

```bash
kubectl apply -f deploy/k8s/base/namespace.yaml
```

## 4. Deployment

`deploy/k8s/base/deployment.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-cicd-lab
  namespace: go-cicd-lab
  labels:
    app.kubernetes.io/name: go-cicd-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: go-cicd-lab
  template:
    metadata:
      labels:
        app.kubernetes.io/name: go-cicd-lab
    spec:
      containers:
        - name: api
          image: go-cicd-lab:local
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
```

重点：

- `replicas: 2` 表示两个 Pod 副本。
- `selector.matchLabels` 必须匹配 Pod template labels。
- `imagePullPolicy: IfNotPresent` 适合 kind 本地镜像。

## 5. Service

`deploy/k8s/base/service.yaml`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: go-cicd-lab
  namespace: go-cicd-lab
  labels:
    app.kubernetes.io/name: go-cicd-lab
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: go-cicd-lab
  ports:
    - name: http
      port: 80
      targetPort: http
```

Service 的 `selector` 会选中 Deployment 创建的 Pod。

`targetPort: http` 对应容器端口名字：

```yaml
ports:
  - name: http
    containerPort: 8080
```

## 6. 应用所有资源

```bash
kubectl apply -f deploy/k8s/base/
```

查看：

```bash
kubectl -n go-cicd-lab get deploy
kubectl -n go-cicd-lab get pods
kubectl -n go-cicd-lab get svc
```

等待 rollout：

```bash
kubectl -n go-cicd-lab rollout status deployment/go-cicd-lab
```

## 7. 本地访问

使用 port-forward：

```bash
kubectl -n go-cicd-lab port-forward service/go-cicd-lab 8080:80
```

另一个终端：

```bash
curl -f http://127.0.0.1:8080/healthz
```

如果服务还没有 `/healthz`，先访问已有路径，或先补健康检查接口。

## 8. 更新镜像

```bash
kubectl -n go-cicd-lab set image deployment/go-cicd-lab api=go-cicd-lab:local
kubectl -n go-cicd-lab rollout status deployment/go-cicd-lab
```

后面 Helm 会用 values 管理镜像。

## 9. 删除资源

```bash
kubectl delete -f deploy/k8s/base/
```

或者删除 namespace：

```bash
kubectl delete namespace go-cicd-lab
```

删除 namespace 会删除该 namespace 下所有资源，谨慎使用。

## 10. 小练习

完成：

1. 创建 namespace、deployment、service。
2. apply 到 kind。
3. 查看 Pod 是否 Running。
4. port-forward 访问服务。
5. 把 replicas 改成 3，再 apply。
6. 观察 Pod 数量变化。

## 11. 本节小结

你现在应该理解：

- Deployment 管理 Pod 副本。
- Service 通过 selector 找到 Pod。
- label 是 Kubernetes 资源关联的关键。
- port-forward 是本地调试访问服务的简单方式。

