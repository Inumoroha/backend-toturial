# 1. 从后端视角理解 Kubernetes

本节目标：学完后你能用后端开发者熟悉的语言，解释 Kubernetes 在项目部署中解决什么问题。

## 简短引入

写 Go 后端时，我们最关心的是接口能不能稳定提供服务、配置能不能按环境切换、日志能不能查、发布失败能不能回滚。Kubernetes 可以理解为一套帮助你运行后端服务的集群操作系统。

## 一、为什么需要它

一个真实后端项目通常不是只运行一个进程。常见情况是：

- API 服务需要多个副本。
- 配置要区分开发、测试、生产。
- 某个进程挂了要自动拉起。
- 发布新版本时不能把所有流量一次性切过去。
- 服务之间要通过稳定地址互相访问。
- 定时任务、迁移任务要有独立运行方式。

如果手工在服务器上 `scp` 二进制、`systemctl restart`，早期项目可以用，但团队规模和服务数量上来后，很容易出现环境不一致、回滚困难、故障不可见的问题。

```text
Kubernetes 的核心价值不是让 YAML 变多，而是把“服务如何运行”从人工操作变成可声明、可重复、可回滚的配置。
```

## 二、基本用法

后端服务在 Kubernetes 中最常见的路径是：

1. Go 代码编译成二进制。
2. 二进制打包进 Docker 镜像。
3. Deployment 负责运行多个副本。
4. Service 给副本提供稳定访问地址。
5. ConfigMap 和 Secret 注入配置。
6. Ingress 暴露 HTTP 入口。

最小资源关系可以先这样理解：

```text
用户请求 -> Ingress -> Service -> Pod -> Go 进程
```

Pod 不是业务服务本身，而是运行容器的最小调度单元。Deployment 不是进程，而是负责管理 Pod 副本的控制器。Service 不是网关，而是给一组 Pod 提供稳定访问入口。

## 三、关键参数/语法/代码结构

你会反复看到这些字段：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-api
  template:
    metadata:
      labels:
        app: user-api
    spec:
      containers:
        - name: user-api
          image: user-api:1.0.0
          ports:
            - containerPort: 8080
```

可以先记住：

- `kind` 表示你要创建什么资源。
- `metadata.name` 是资源名称。
- `replicas` 是副本数。
- `selector` 用来找到归这个 Deployment 管理的 Pod。
- `template` 是 Pod 模板。
- `image` 是后端服务镜像。

## 四、真实后端场景示例

假设你有一个用户服务 `user-api`，本地运行命令是：

```bash
./user-api
```

迁移到 Kubernetes 后，不再直接登录服务器启动它，而是把运行方式写进 YAML：

```yaml
containers:
  - name: user-api
    image: registry.example.com/backend/user-api:2026.07.06
    env:
      - name: APP_ENV
        value: "prod"
    ports:
      - containerPort: 8080
```

真实项目中，这样做的好处是发布记录清楚：你能知道生产环境跑的是哪个镜像、几个副本、暴露哪个端口、用了哪些配置。

## 五、注意点

- Kubernetes 不替你修复代码里的 panic、慢 SQL、事务错误。
- Kubernetes 不等于微服务，单体服务也可以部署在 Kubernetes。
- Kubernetes 适合管理无状态服务，但数据库、消息队列这类有状态组件需要更谨慎。
- 初学阶段先把一个服务部署稳，不要急着上 Service Mesh、复杂网关和多集群。

## 六、常见误区

- 误区：学 Kubernetes 就是背资源对象。
  真实项目更看重你能不能部署、观察、回滚。
- 误区：所有东西都应该放进 Kubernetes。
  数据库是否放入集群要看团队运维能力和数据可靠性要求。
- 误区：YAML 能 apply 成功就算部署成功。
  后端服务还要确认健康检查、日志、配置、访问路径都正常。

## 七、本节达标标准

- 能解释 Pod、Deployment、Service 的基本关系。
- 能说清楚 Kubernetes 对后端发布和运维的价值。
- 能理解“声明式配置”的直觉。
- 知道初学阶段不要过早深入复杂底层原理。

## 八、零基础加餐：用一次故障理解“控制器”

先把 Deployment 想成一名持续巡检的管理员。你声明 `replicas: 2`，它就会不断比较“期望有 2 个 Pod”和“现在实际有几个 Pod”。少了一个，它会创建新的；多了一个，它会逐步收敛到期望数量。

后面搭好集群后，可以回到这里做实验：

```bash
kubectl create deployment controller-demo --image=nginx:alpine --replicas=2
kubectl get pods -l app=controller-demo
kubectl delete pod <其中一个-pod-name>
kubectl get pods -l app=controller-demo -w
```

预期现象：旧 Pod 进入终止状态，同时出现一个名字不同的新 Pod。按 `Ctrl+C` 结束观察，再清理：

```bash
kubectl delete deployment controller-demo
```

自测：为什么删除 Pod 后它会回来，而删除 Deployment 后不会回来？因为 Pod 是 Deployment 管理的实际对象，而 Deployment 才保存副本的期望状态。
