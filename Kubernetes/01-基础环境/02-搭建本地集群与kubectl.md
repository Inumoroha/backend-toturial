# 2. 搭建本地集群与 kubectl

本节目标：学完后你能在本机启动一个 Kubernetes 集群，并用 `kubectl` 查看和操作资源。

## 简短引入

后端开发者学习 Kubernetes，第一步不是买云服务器，而是在本地搭一个能反复试错的环境。真实项目中的部署问题，很多都可以先在本地小集群里复现。

## 一、为什么需要它

本地集群的作用类似后端项目里的本地数据库和本地 Redis：

- 让你低成本练习部署。
- 让你验证 YAML 是否正确。
- 让你观察 Pod 重启、Service 访问、日志查询。
- 让你在不影响别人环境的情况下踩坑。

常见选择：

- Docker Desktop Kubernetes：适合已经使用 Docker Desktop 的 Windows/macOS 用户。
- Minikube：学习资料多，适合初学。
- kind：用 Docker 容器模拟 Kubernetes 节点，适合 CI 和脚本化。

本教程后续示例只依赖标准 `kubectl`，你用哪种本地集群都可以。

## 二、基本用法

安装好本地集群后，先确认 `kubectl` 能连上集群。

Windows PowerShell：

```powershell
kubectl version
kubectl cluster-info
kubectl get nodes
```

Linux/macOS：

```bash
kubectl version
kubectl cluster-info
kubectl get nodes
```

如果你用 Minikube：

Windows PowerShell：

```powershell
minikube start
kubectl get nodes
```

Linux/macOS：

```bash
minikube start
kubectl get nodes
```

如果你用 kind：

Windows PowerShell：

```powershell
kind create cluster --name backend-lab
kubectl cluster-info --context kind-backend-lab
kubectl get nodes
```

Linux/macOS：

```bash
kind create cluster --name backend-lab
kubectl cluster-info --context kind-backend-lab
kubectl get nodes
```

## 三、关键参数/语法/代码结构

`kubectl` 是你和 Kubernetes API Server 交互的命令行工具。常见命令先掌握这些：

```bash
kubectl get pods
kubectl get deploy
kubectl get svc
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl apply -f app.yaml
kubectl delete -f app.yaml
```

真实项目中的作用：

- `get` 用来快速看资源是否存在、状态是否正常。
- `describe` 用来看事件和调度原因。
- `logs` 用来看应用输出。
- `apply` 用来把 YAML 变更提交给集群。
- `delete` 用来删除资源，学习环境常用，生产环境要谨慎。

## 四、真实后端场景示例

先运行一个临时的 HTTP 服务，模拟后端 API。

```bash
kubectl create deployment hello-api --image=hashicorp/http-echo -- -text=hello-k8s
kubectl expose deployment hello-api --port=8080 --target-port=5678
kubectl get pods
kubectl get svc
```

本地转发端口访问：

Windows PowerShell：

```powershell
kubectl port-forward svc/hello-api 8080:8080
```

另开一个 PowerShell：

```powershell
curl.exe http://127.0.0.1:8080
```

Linux/macOS：

```bash
kubectl port-forward svc/hello-api 8080:8080
curl http://127.0.0.1:8080
```

清理资源：

```bash
kubectl delete svc hello-api
kubectl delete deploy hello-api
```

这个例子在真实项目中的意义是：先验证“服务能跑、能被 Service 选择、能通过端口转发访问”。后续换成你自己的 Go 镜像，本质路径是一样的。

## 五、注意点

- `kubectl` 操作的是当前 context，生产和测试 context 切错很危险。
- `kubectl delete` 在生产环境不要凭感觉执行。
- `port-forward` 适合调试，不是正式暴露服务的方式。
- 本地集群和生产集群能力可能不同，特别是负载均衡、存储、Ingress。

```text
执行 kubectl apply/delete 前，先确认当前 context 和 namespace。
```

## 六、常见错误

- 错误：`kubectl get nodes` 报无法连接。
  通常是本地集群没启动，或 kubeconfig context 不对。
- 错误：Pod 一直 `ImagePullBackOff`。
  多半是镜像名错误、镜像仓库不可访问、没有登录私有仓库。
- 错误：`curl` Service 名称失败。
  在本机不能直接解析集群内 Service DNS，需要 `port-forward` 或 Ingress。

## 七、本节达标标准

- 能启动一个本地 Kubernetes 集群。
- 能用 `kubectl get` 查看节点、Pod、Deployment、Service。
- 能通过 `port-forward` 访问集群内服务。
- 能清理自己创建的练习资源。

