# Go 后端工程师 Kubernetes 学习路线

这套路线面向零基础或正在学习 Go 后端、希望把服务部署到 Kubernetes 的学习者。目标不是背概念，而是能把一个真实后端服务从本地代码逐步交付到集群里，并知道出问题时从哪里查。

如果你连 Pod、镜像、YAML 都没有接触过，先阅读[教程总入口](../README.md)和[零基础术语表与命令速查](./01-零基础术语表与命令速查.md)。它们会解释本教程反复出现的词、命令占位符和统一排障顺序。

官方文档建议随时对照阅读：Kubernetes 官方文档 https://kubernetes.io/docs/ 。本教程不强绑定具体小版本，实践时以你本地工具支持的稳定版本为准。

```text
学习 Kubernetes 的主线不是“记住所有资源对象”，而是理解一个后端服务从镜像、配置、运行、副本、访问、权限、数据任务到发布回滚的完整路径。
```

## 一、学习前置条件

具备下面基础会更顺利，但它们不是开始学习的硬门槛：

- 能写一个简单 Go HTTP 服务，例如用户、文章、订单、短链接接口。
- 理解 Docker 镜像、容器、端口映射、环境变量。
- 会使用 Git、命令行、基本 Linux 文件路径。
- 了解 HTTP、反向代理、数据库连接、日志、配置文件这些后端常识。

不需要提前掌握：

- Kubernetes 源码。
- CNI、CSI、CRI 等底层实现。
- Service Mesh。
- 云厂商托管集群高级能力。

这些内容等你能稳定部署服务后再学更合适。

## 二、阶段路线

### 第 1 阶段：建立后端部署直觉

目标：知道 Kubernetes 解决的不是“写代码”，而是“让服务稳定运行、可访问、可发布、可回滚”。

建议学习：

1. 从后端视角理解 Kubernetes。
2. 搭建本地集群，掌握 `kubectl`。
3. 把 Go 服务打包成镜像。

产出标准：

- 能说清楚 Pod、Deployment、Service 分别在真实项目中承担什么角色。
- 能在本地启动一个集群。
- 能把一个 Go HTTP 服务构建成镜像。

### 第 2 阶段：让服务在集群中跑起来

目标：把一个后端服务稳定运行在 Kubernetes 中，并能被集群内部访问。

建议学习：

1. Pod 与 Deployment。
2. Service 与服务发现。
3. ConfigMap 与 Secret。
4. 健康检查与优雅停机。
5. 资源请求、限制与副本数。

产出标准：

- 能写出一个可运行的 Deployment。
- 能通过 Service 访问服务。
- 能把配置从镜像中剥离出来。
- 能解释为什么生产服务必须配置 readiness probe。

### 第 3 阶段：让服务面向真实环境

目标：服务不只是在集群里跑，还要能被用户访问，并具备基本权限隔离。

建议学习：

1. Ingress 与网关入口。
2. Namespace 与环境隔离。
3. RBAC 与最小权限。

产出标准：

- 能通过域名或本地 Host 访问服务。
- 能用 Namespace 区分 dev、test、prod。
- 能给应用只分配必要权限。

### 第 4 阶段：处理数据、迁移和定时任务

目标：理解后端项目里数据库迁移、定时任务、临时任务在 Kubernetes 中怎么落地。

建议学习：

1. PersistentVolume 与有状态依赖。
2. Job 与数据库迁移。
3. CronJob 与定时任务。

产出标准：

- 知道数据库通常不建议初学阶段直接放进生产 Kubernetes。
- 能用 Job 跑一次性迁移任务。
- 能用 CronJob 执行订单超时、日志清理这类定时任务。

### 第 5 阶段：工程化交付与排障

目标：掌握真实团队里更常见的部署组织方式和问题定位方式。

建议学习：

1. Helm 模板化部署。
2. Kustomize 多环境配置。
3. 日志、事件与故障排查。
4. 滚动发布、回滚与生产检查。

产出标准：

- 能把重复 YAML 整理成 Helm Chart 或 Kustomize overlay。
- 能用 `kubectl logs`、`describe`、`events` 定位常见问题。
- 能进行滚动发布和回滚。

### 第 6 阶段：完整项目落地

目标：把前面知识串成一个小型但真实的后端项目部署流程。

建议项目：

- 短链接服务。
- 用户权限服务。
- 订单库存服务。
- 文章发布服务。

本教程使用“短链接服务”作为主项目，因为它足够贴近后端业务，又不会被复杂业务规则淹没。

产出标准：

- 能部署 API 服务。
- 能管理配置和密钥。
- 能执行数据库迁移。
- 能暴露入口。
- 能查看日志和回滚。

### 第 7 阶段：生产进阶补齐

目标：补齐真实团队常问的访问边界、监控告警和 CI/CD 发布流程。

建议学习：

1. NetworkPolicy 与服务访问边界。
2. 监控指标与告警。
3. CI/CD 与镜像发布流水线。

产出标准：

- 能限制只有指定服务可以访问数据库。
- 能区分日志、指标和事件各自解决什么问题。
- 能设计一条从代码提交到 Kubernetes 发布的保守流水线。

## 三、推荐学习顺序

按下面编号阅读即可：

1. `01-基础环境/01-从后端视角理解Kubernetes.md`
2. `01-基础环境/02-搭建本地集群与kubectl.md`
3. `01-基础环境/03-把Go服务打包成镜像.md`
4. `02-工作负载/04-Pod与Deployment.md`
5. `02-工作负载/05-Service与服务发现.md`
6. `02-工作负载/06-ConfigMap与Secret.md`
7. `02-工作负载/07-探针与优雅停机.md`
8. `02-工作负载/08-资源请求限制与扩缩容.md`
9. `03-访问与安全/09-Ingress与后端入口.md`
10. `03-访问与安全/10-Namespace与环境隔离.md`
11. `03-访问与安全/11-RBAC与最小权限.md`
12. `04-数据与任务/12-PersistentVolume与有状态依赖.md`
13. `04-数据与任务/13-Job与数据库迁移.md`
14. `04-数据与任务/14-CronJob与定时任务.md`
15. `05-交付与排障/15-Helm模板化部署.md`
16. `05-交付与排障/16-Kustomize多环境配置.md`
17. `05-交付与排障/17-日志事件与故障排查.md`
18. `05-交付与排障/18-滚动发布回滚与生产检查.md`
19. `06-项目实战/19-短链接服务Kubernetes落地项目.md`
20. `07-生产进阶/20-NetworkPolicy与服务访问边界.md`
21. `07-生产进阶/21-监控指标与告警.md`
22. `07-生产进阶/22-CICD与镜像发布流水线.md`

## 四、学习方法

每学一节都建议做到三件事：

- 自己复制命令跑一遍。
- 改一个参数，观察现象。
- 记录一个错误和解决过程。

零基础学习者建议每次只学一章，控制在 45–90 分钟。前 20 分钟读概念，中间亲手运行示例，最后不看答案完成章末自测。命令报错时先保留输出，不要立刻删除 Pod 或重建集群。

建议为整套教程创建一个练习目录：

```text
kubernetes-lab/
  chapter-03-image/
  chapter-04-deployment/
  chapter-05-service/
  notes/
```

每章的 YAML 独立保存。这样出错时能知道资源来自哪一章，也方便用 Git 比较自己改了什么。

```text
Kubernetes 不是靠“看懂 YAML”学会的，而是靠“改配置、观察状态、定位问题、恢复服务”学会的。
```

## 五、生产实践保守原则

- 不要把数据库密码写进镜像或 Git 仓库。
- 不要在生产环境随手使用 `latest` 镜像标签。
- 不要忽略 readiness probe，否则滚动发布时可能把未就绪的新 Pod 接入流量。
- 不要给应用默认管理员权限。
- 不要在没有回滚方案时做生产发布。
- 数据库迁移要可回滚，至少要可停止、可重试、可观察。
- 资源限制要从监控数据出发，不要靠猜。

## 六、最终达标标准

学完这套路线上手一个后端项目时，你应该能够：

- 把 Go API 服务构建成镜像。
- 编写 Deployment、Service、ConfigMap、Secret。
- 通过 Ingress 暴露 HTTP 服务。
- 使用 Job 执行数据库迁移。
- 使用 CronJob 执行业务定时任务。
- 使用 Namespace 和 RBAC 做基本隔离。
- 使用 Helm 或 Kustomize 管理多环境配置。
- 定位 CrashLoopBackOff、ImagePullBackOff、Service 访问失败、配置错误等常见问题。
- 制定一次可回滚的发布流程。
- 使用 NetworkPolicy 做基本服务访问控制。
- 为后端服务定义核心指标和告警。
- 设计一条可审计、可回滚的 CI/CD 发布流水线。

## 七、官方文档对照阅读

这套教程按后端项目落地顺序组织，不按官方文档目录顺序展开。学习时可以把下面页面当作权威对照：

- Kubernetes 文档首页：https://kubernetes.io/docs/
- Deployment：https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- Service：https://kubernetes.io/docs/concepts/services-networking/service/
- Ingress：https://kubernetes.io/docs/concepts/services-networking/ingress/
- ConfigMap：https://kubernetes.io/docs/concepts/configuration/configmap/
- Secret：https://kubernetes.io/docs/concepts/configuration/secret/
- Liveness、Readiness、Startup Probes：https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/
- Job：https://kubernetes.io/docs/concepts/workloads/controllers/job/
- CronJob：https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/
- RBAC：https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- NetworkPolicy：https://kubernetes.io/docs/concepts/services-networking/network-policies/

```text
先用本教程建立“后端项目怎么落地”的路径，再用官方文档查字段细节和版本差异。
```
