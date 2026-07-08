# 阶段 6：Kubernetes 与 Helm

> 本阶段目标：把第 5 阶段的“单机 Docker Compose 部署”迁移到 Kubernetes，并用 Helm 管理 Go 后端服务的多环境部署、升级和回滚。

## 学习顺序

请按下面顺序学习：

1. [01-kubernetes-mental-model.md](./01-kubernetes-mental-model.md)
   - 建立 Kubernetes 心智模型。
   - 理解声明式 API、期望状态、控制器、Pod、Node、Namespace。

2. [02-local-cluster-and-kubectl.md](./02-local-cluster-and-kubectl.md)
   - 使用 kind 或 minikube 启动本地 Kubernetes。
   - 学习 kubectl 上下文、namespace 和基础命令。

3. [03-first-manifests-namespace-deployment-service.md](./03-first-manifests-namespace-deployment-service.md)
   - 编写第一个 Namespace、Deployment、Service。
   - 把 Go 镜像运行在 Kubernetes 中。

4. [04-configmap-secret-and-image-pull-secret.md](./04-configmap-secret-and-image-pull-secret.md)
   - 学习 ConfigMap、Secret、imagePullSecrets。
   - 把环境配置和敏感信息从镜像中分离。

5. [05-probes-resources-rollout-and-rollback.md](./05-probes-resources-rollout-and-rollback.md)
   - 学习 readiness/liveness/startup probes。
   - 学习 requests/limits、滚动更新、rollout status、rollback。

6. [06-service-ingress-and-local-access.md](./06-service-ingress-and-local-access.md)
   - 学习 ClusterIP、NodePort、LoadBalancer、Ingress。
   - 本地用 port-forward 或 Ingress 访问 Go 服务。

7. [07-jobs-migrations-and-operational-tasks.md](./07-jobs-migrations-and-operational-tasks.md)
   - 学习 Job/CronJob。
   - 理解数据库迁移在 Kubernetes 中更适合用独立 Job。

8. [08-helm-chart-basics.md](./08-helm-chart-basics.md)
   - 学习 Helm Chart 结构。
   - 创建 `Chart.yaml`、`values.yaml`、templates。

9. [09-helm-values-and-multi-environment.md](./09-helm-values-and-multi-environment.md)
   - 学习 values 分层。
   - 使用 `values-staging.yaml`、`values-production.yaml` 管理环境差异。

10. [10-helm-install-upgrade-rollback-and-history.md](./10-helm-install-upgrade-rollback-and-history.md)
    - 学习 `helm lint`、`helm template`、`helm upgrade --install`、`helm rollback`。
    - 理解 Helm release 和 revision。

11. [11-ci-validation-and-deploy-with-helm.md](./11-ci-validation-and-deploy-with-helm.md)
    - 在 CI 中校验 Kubernetes manifests 和 Helm Chart。
    - 使用 GitHub Actions 部署到 Kubernetes。

12. [12-practice-deploy-go-cicd-lab-to-kubernetes.md](./12-practice-deploy-go-cicd-lab-to-kubernetes.md)
    - 为 `go-cicd-lab` 落地 Kubernetes manifests 和 Helm Chart。
    - 输出 `docs/kubernetes.md`。

13. [13-troubleshooting-and-kubectl-playbook.md](./13-troubleshooting-and-kubectl-playbook.md)
    - 学习 Pod 启动失败、镜像拉取失败、Service 不通、probe 失败、Helm 升级失败的排查方法。

14. [14-review-checklist-and-quiz.md](./14-review-checklist-and-quiz.md)
    - 用清单和自测题检查是否可以进入第 7 阶段。

## 本阶段建议时间

- 快速学习：1 到 2 周。
- 扎实学习：2 到 3 周。
- 推荐方式：先手写 Kubernetes YAML，再用 Helm 模板化。不要一上来只复制 Chart。

## 本阶段你要准备什么

- 已完成第 4 阶段：有可拉取的 Go 服务镜像。
- 已完成第 5 阶段：理解环境、配置、健康检查、回滚。
- 本地安装 Docker。
- 安装 kubectl。
- 安装 kind 或 minikube。
- 安装 Helm。

检查：

```bash
kubectl version --client
kind version
minikube version
helm version
docker version
```

## 学完后你应该能做到

- 用 Deployment 和 Service 运行 Go 服务。
- 使用 ConfigMap 和 Secret 注入配置。
- 使用 imagePullSecrets 拉取私有镜像。
- 配置 readiness/liveness/startup probes。
- 配置资源 requests/limits。
- 用 kubectl 查看日志、事件、rollout 状态。
- 用 Helm 管理多环境部署。
- 用 Helm 回滚 release。
- 在 CI 中校验 Helm Chart。

## 推荐官方资料

- Kubernetes Pods：<https://kubernetes.io/docs/concepts/workloads/pods/>
- Kubernetes Deployments：<https://kubernetes.io/docs/concepts/workloads/controllers/deployment/>
- Kubernetes Services：<https://kubernetes.io/docs/concepts/services-networking/service/>
- Kubernetes Ingress：<https://kubernetes.io/docs/concepts/services-networking/ingress/>
- Kubernetes ConfigMaps：<https://kubernetes.io/docs/concepts/configuration/configmap/>
- Kubernetes Secrets：<https://kubernetes.io/docs/concepts/configuration/secret/>
- Kubernetes Probes：<https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/>
- kubectl Quick Reference：<https://kubernetes.io/docs/reference/kubectl/quick-reference/>
- kind Quick Start：<https://kind.sigs.k8s.io/docs/user/quick-start/>
- minikube Start：<https://minikube.sigs.k8s.io/docs/start/>
- Helm Docs：<https://helm.sh/docs/>

