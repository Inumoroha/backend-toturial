# 01：Kubernetes 心智模型

## 1. 本节目标

第 5 阶段我们用 Docker Compose 在单台 VM 上部署。

第 6 阶段进入 Kubernetes。

你要先建立这张图：

```text
你提交 YAML
-> Kubernetes API Server 保存期望状态
-> 控制器持续观察实际状态
-> 调度器把 Pod 放到 Node
-> kubelet 在 Node 上运行容器
-> Service 提供稳定访问入口
```

## 2. Kubernetes 解决什么问题

Kubernetes 主要解决：

- 多副本运行。
- 容器调度。
- 服务发现。
- 滚动更新。
- 自动恢复。
- 配置和密钥注入。
- 资源管理。
- 声明式部署。

它不是简单的“更复杂 Docker Compose”。

它是一个通过 API 管理集群状态的平台。

## 3. 声明式 API

在 Compose 中你写：

```bash
docker compose up -d
```

在 Kubernetes 中你写：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-cicd-lab
spec:
  replicas: 2
```

然后：

```bash
kubectl apply -f deployment.yaml
```

你声明期望状态：

```text
我希望有 2 个 go-cicd-lab 副本。
```

Kubernetes 控制器负责让实际状态靠近期望状态。

## 4. 核心对象

| 对象 | 作用 |
| --- | --- |
| Cluster | Kubernetes 集群 |
| Node | 运行 Pod 的机器 |
| Pod | Kubernetes 中最小可部署单元 |
| Deployment | 管理无状态应用副本和滚动更新 |
| ReplicaSet | 维持 Pod 副本数量，通常由 Deployment 管 |
| Service | 给一组 Pod 提供稳定访问入口 |
| Ingress | 基于域名/路径暴露 HTTP/HTTPS |
| ConfigMap | 非敏感配置 |
| Secret | 敏感配置 |
| Namespace | 资源隔离边界 |
| Job | 一次性任务 |
| CronJob | 定时任务 |

## 5. Pod 是什么

Pod 是 Kubernetes 中最小可部署对象。

通常一个 Pod 里放一个主容器。

一个 Pod 有：

- IP。
- 容器。
- volume。
- env。
- probes。
- resource requests/limits。

重要：Pod 不是长期稳定身份。

Pod 可能因为重启、更新、节点故障而被替换。

所以不要让外部系统直接依赖 Pod IP。

## 6. Deployment 是什么

Deployment 管理一组 Pod。

它负责：

- 创建 Pod。
- 保持副本数量。
- 滚动更新。
- 回滚。
- 通过 ReplicaSet 管理版本。

Go API 这类无状态服务，通常使用 Deployment。

## 7. Service 是什么

Pod 会变，Pod IP 也会变。

Service 提供稳定访问入口。

Service 通过 label selector 找到后端 Pod：

```yaml
selector:
  app: go-cicd-lab
```

只要 Pod 有匹配 label，就会成为 Service 后端。

## 8. Namespace 是什么

Namespace 用于在一个集群中隔离资源。

常见：

```text
go-cicd-lab-staging
go-cicd-lab-production
```

学习阶段可以为项目创建一个 namespace。

生产中，namespace 不是强安全边界，但它是资源组织和权限管理的重要基础。

## 9. 控制器循环

Kubernetes 的核心思想是控制循环：

```text
观察实际状态
-> 和期望状态比较
-> 执行动作缩小差距
-> 重复
```

例如你声明：

```text
replicas: 3
```

如果实际只有 2 个 Pod，Deployment controller 会创建新 Pod。

如果实际有 4 个 Pod，它会删除一个。

## 10. 和 Docker Compose 的差异

| 对比 | Docker Compose | Kubernetes |
| --- | --- | --- |
| 运行范围 | 单机为主 | 集群 |
| 部署方式 | 命令式偏多 | 声明式 |
| 服务发现 | Compose 网络名 | Service/DNS |
| 更新 | `compose up` | Deployment rollout |
| 回滚 | 依赖脚本 | rollout/Helm revision |
| 配置 | env/env_file | ConfigMap/Secret |
| 入口 | ports/reverse proxy | Service/Ingress |

## 11. 小练习

用自己的话回答：

1. Pod 和容器有什么关系？
2. Deployment 为什么比直接创建 Pod 更常用？
3. Service 为什么需要 selector？
4. Namespace 解决什么问题？
5. Kubernetes 中“期望状态”是什么意思？

## 12. 本节小结

你现在应该理解：

- Kubernetes 是声明式系统。
- Pod 是最小可部署单元，但通常不直接管理 Pod。
- Deployment 管理副本和滚动更新。
- Service 给变化的 Pod 提供稳定入口。
- 控制器持续让实际状态靠近期望状态。

