# 02：本地集群与 kubectl

## 1. 本节目标

这一节准备本地 Kubernetes 环境。

你要能：

- 创建本地集群。
- 使用 kubectl 查看集群。
- 切换 context。
- 创建 namespace。
- 加载本地镜像。

## 2. kind 和 minikube 怎么选

推荐：

- 想轻量、适合 CI、本地测试：kind。
- 想要内置 dashboard、addons、接近个人学习体验：minikube。

本教程主线使用 kind，minikube 也可以。

## 3. 创建 kind 集群

安装 kind 后：

```bash
kind create cluster --name go-cicd-lab
```

查看：

```bash
kubectl cluster-info
kubectl get nodes
kubectl config current-context
```

删除：

```bash
kind delete cluster --name go-cicd-lab
```

## 4. kind 加载本地镜像

如果你本地构建镜像：

```bash
docker build -t go-cicd-lab:local .
```

加载到 kind：

```bash
kind load docker-image go-cicd-lab:local --name go-cicd-lab
```

这样 Kubernetes Pod 可以使用：

```text
go-cicd-lab:local
```

并设置：

```yaml
imagePullPolicy: IfNotPresent
```

## 5. minikube 基础命令

如果你使用 minikube：

```bash
minikube start
kubectl get nodes
minikube dashboard
```

使用本地 Docker daemon 构建给 minikube：

```bash
eval $(minikube docker-env)
docker build -t go-cicd-lab:local .
```

Windows PowerShell 使用 minikube docker-env 的方式要按 minikube 输出提示执行。

## 6. kubectl context

查看 contexts：

```bash
kubectl config get-contexts
```

当前 context：

```bash
kubectl config current-context
```

切换：

```bash
kubectl config use-context kind-go-cicd-lab
```

使用真实集群前，一定确认 context，避免把学习 YAML apply 到生产集群。

## 7. Namespace

创建 namespace：

```bash
kubectl create namespace go-cicd-lab
```

查看：

```bash
kubectl get namespaces
```

设置当前 context 默认 namespace：

```bash
kubectl config set-context --current --namespace=go-cicd-lab
```

也可以每次命令加：

```bash
kubectl -n go-cicd-lab get pods
```

## 8. kubectl 基础命令

查看资源：

```bash
kubectl get pods
kubectl get deploy
kubectl get svc
kubectl get all
```

详细信息：

```bash
kubectl describe pod <pod-name>
```

日志：

```bash
kubectl logs <pod-name>
```

进入容器：

```bash
kubectl exec -it <pod-name> -- sh
```

注意：distroless 镜像通常没有 shell。

应用 YAML：

```bash
kubectl apply -f k8s/
```

删除：

```bash
kubectl delete -f k8s/
```

## 9. 小练习

完成：

1. 创建 kind 集群。
2. 查看 node。
3. 创建 namespace。
4. 设置默认 namespace。
5. 构建本地 Go 镜像。
6. 加载镜像到 kind。

命令参考：

```bash
kind create cluster --name go-cicd-lab
kubectl get nodes
kubectl create namespace go-cicd-lab
kubectl config set-context --current --namespace=go-cicd-lab
docker build -t go-cicd-lab:local .
kind load docker-image go-cicd-lab:local --name go-cicd-lab
```

## 10. 本节小结

你现在应该理解：

- kind 和 minikube 都能提供本地 Kubernetes。
- kubectl 通过 context 操作不同集群。
- namespace 用来组织资源。
- 本地镜像需要加载到 kind 或构建到 minikube 环境中。

