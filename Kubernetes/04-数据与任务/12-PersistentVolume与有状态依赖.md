# 12. PersistentVolume 与有状态依赖

本节目标：学完后你能理解 Kubernetes 中持久化存储的基本用法，并知道数据库上集群的风险。

## 简短引入

后端服务本身通常是无状态的，但数据库、文件、消息队列不是。Pod 被删除后，容器文件系统也可能消失。持久化数据要用 Volume 和 PersistentVolumeClaim。

## 一、为什么需要它

真实项目中这些数据不能随 Pod 消失：

- PostgreSQL 数据目录。
- 用户上传文件。
- 消息队列持久化数据。
- 本地缓存索引。

但要注意：能持久化不代表适合把数据库随便放进 Kubernetes。生产数据库涉及备份、恢复、主从、高可用、性能和运维经验。

```text
初学阶段可以在 Kubernetes 中练习 PostgreSQL，但生产环境优先选择团队能稳定运维的数据库方案。
```

## 二、基本用法

创建 PVC：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

PostgreSQL Deployment 片段：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          env:
            - name: POSTGRES_USER
              value: app
            - name: POSTGRES_PASSWORD
              value: password
            - name: POSTGRES_DB
              value: short
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: postgres-data
```

创建 Service：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

## 三、关键参数/语法/代码结构

- PVC：应用声明需要多大存储。
- PV：实际存储资源，可能由集群动态创建。
- StorageClass：存储类型，云盘、本地盘等由它决定。
- `ReadWriteOnce`：通常表示一个节点读写挂载。
- `mountPath`：容器内挂载路径。

学习环境里，默认 StorageClass 可能已经存在。查看：

```bash
kubectl get storageclass
kubectl get pvc
```

## 四、真实后端场景示例

短链接服务连接 PostgreSQL：

```text
postgres://app:password@postgres:5432/short?sslmode=disable
```

这里的 `postgres` 是 Service 名称。业务服务不关心 PostgreSQL Pod IP，只关心稳定 Service 地址。

真实项目中，如果 PostgreSQL 是云数据库，Kubernetes 中就不需要 PostgreSQL Deployment，只需要把连接地址放进 Secret。

## 五、注意点

- Deployment 运行数据库只适合学习或简单环境，生产更常见 StatefulSet 或托管数据库。
- 数据库密码不要明文写在正式 YAML。
- 删除 PVC 可能导致数据丢失，生产必须确认回收策略。
- 挂载本地存储会影响 Pod 调度，节点坏了恢复会复杂。

## 六、常见错误

- 错误：以为 Pod 删除后数据还一定在。
  没有 PVC 或存储回收策略不清楚时，数据可能丢失。
- 错误：数据库使用多个副本 Deployment。
  普通 PostgreSQL 镜像不能靠简单 replicas 实现高可用。
- 错误：把用户上传文件存在容器本地目录。
  Pod 重建后文件可能消失，应使用对象存储或持久卷。

## 七、本节达标标准

- 能解释 PVC 的作用。
- 能为 PostgreSQL 练习环境配置持久化目录。
- 能说明生产数据库放进 Kubernetes 的主要风险。
- 能区分学习环境和生产环境的保守选择。

## 八、观察卷的生命周期

先分清三个对象：StorageClass 描述“如何提供存储”；PVC 是应用提出的存储申请；PV 是满足申请的实际卷。Pod 通常引用 PVC，而不是直接挑选某个 PV。

```bash
kubectl get storageclass
kubectl get pvc
kubectl get pv
kubectl describe pvc <pvc-name>
```

PVC 应从 `Pending` 变为 `Bound`。一直 `Pending` 时查看 Events，常见原因是没有默认 StorageClass、访问模式不支持或请求容量无法满足。

实验：创建使用 PVC 的 Pod，在挂载目录写一个文本文件；删除并重建 Pod 后再次读取。文件仍在才说明数据生命周期独立于 Pod。随后查看 PV 的 `persistentVolumeReclaimPolicy`，理解删除 PVC 后数据是保留还是删除。

这项实验只证明“卷能持久化”，不证明数据库已经高可用。数据库还需要一致性、备份恢复、复制、故障切换、容量和升级策略。
