# 4. Pod 与 Deployment

本节目标：学完后你能用 Deployment 在 Kubernetes 中运行 Go 后端服务，并理解副本和滚动更新的基本作用。

## 简短引入

后端服务最终要变成一个或多个运行中的进程。在 Kubernetes 里，容器运行在 Pod 中，而生产项目通常不会直接手写裸 Pod，而是用 Deployment 管理 Pod。

## 一、为什么需要它

直接运行一个 Pod 类似手工启动一个进程。进程挂了你要处理，版本更新你要处理，副本扩缩你也要处理。Deployment 的作用是声明你希望服务保持什么状态。

常见场景：

- 用户服务需要 2 个副本。
- 订单服务发布新镜像。
- 某个 Pod 异常退出后自动重建。
- 更新失败时回滚到上一个版本。

## 二、基本用法

创建 `deployment.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: short-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: short-api
  template:
    metadata:
      labels:
        app: short-api
    spec:
      containers:
        - name: short-api
          image: short-api:0.1.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
```

应用配置：

```bash
kubectl apply -f deployment.yaml
kubectl get deploy
kubectl get pods -l app=short-api
```

如果你用 kind，本地镜像需要加载进集群：

```bash
kind load docker-image short-api:0.1.0 --name backend-lab
```

如果你用 Minikube，可以先切到 Minikube 的 Docker 环境再构建，或使用：

```bash
minikube image load short-api:0.1.0
```

## 三、关键参数/语法/代码结构

- `replicas`：期望副本数。真实项目要结合流量和资源来定。
- `selector.matchLabels`：Deployment 用它匹配要管理的 Pod。
- `template.metadata.labels`：Pod 标签，必须和 selector 对上。
- `imagePullPolicy`：本地学习可用 `IfNotPresent`，生产通常配合明确镜像标签。
- `containerPort`：容器内服务端口，方便 Service 和读者理解。

```text
selector 一旦创建后不要随意改。改错后 Deployment 可能找不到自己的 Pod。
```

## 四、真实后端场景示例

短链接服务发布了新版本 `short-api:0.2.0`，你可以更新镜像：

```bash
kubectl set image deployment/short-api short-api=short-api:0.2.0
kubectl rollout status deployment/short-api
kubectl rollout history deployment/short-api
```

如果发现新版本创建短链接接口报错，可以回滚：

```bash
kubectl rollout undo deployment/short-api
kubectl rollout status deployment/short-api
```

在真实项目中，这个流程通常由 CI/CD 执行，但后端工程师要知道它背后发生了什么。

## 五、注意点

- 生产服务不要只有 1 个副本，除非业务允许短暂不可用。
- 副本数增加不代表数据库压力会自动下降，反而可能增加连接数。
- Go 服务要正确处理 SIGTERM，否则滚动更新时请求可能被中断。
- 镜像拉取失败不是 Kubernetes 代码问题，通常是镜像名、仓库权限或网络问题。

## 六、常见错误

- 错误：`replicas: 2` 但只有一个 Pod Ready。
  用 `kubectl describe pod` 查看事件，常见原因是镜像拉取失败或端口启动失败。
- 错误：修改 YAML 后没变化。
  确认执行了 `kubectl apply -f deployment.yaml`，以及改的是正确 namespace。
- 错误：直接删除 Pod 当作发布。
  Deployment 会重建 Pod，但镜像版本不变，这不是可靠发布方式。

## 七、本节达标标准

- 能编写 Deployment 运行 Go 服务。
- 能查看 Deployment 和 Pod 状态。
- 能更新镜像并观察滚动更新。
- 能执行一次回滚。

