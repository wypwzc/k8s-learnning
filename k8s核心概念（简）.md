kubernetes（k8s）的核心概念，并按照类别进行分类，帮助你形成清晰的结构化认知。

------

## **一、集群结构与资源层级**

1. 集群（Cluster）
   - 定义：由一组节点（Node）组成，运行容器化应用的整体环境。
   - 组成：控制平面（Control Plane）+ 工作节点（Worker Node）。
2. 节点（Node）
   - 定义：运行 Pod 的机器，可以是物理机或虚拟机。
   - 类型：
     - **Master/Control Plane Node**：管理集群的状态。
     - **Worker Node**：运行应用容器。
   - 核心组件：kubelet、kube-proxy、容器运行时（Docker/containerd）。

------

## **二、工作负载（Workload）资源**

用于运行应用程序的实体。

1. Pod
   - 最小可调度单元，一个或多个容器共享网络和存储。
   - 特点：临时性，生命周期短。
2. ReplicaSet（RS）
   - 确保 Pod 副本数量维持在指定数量。
   - 通常由 Deployment 管理。
3. Deployment
   - 管理 Pod 和 ReplicaSet，提供滚动更新、回滚。
   - 核心用于无状态服务（stateless）。
4. StatefulSet
   - 用于有状态服务（stateful），保证 Pod 的顺序和稳定的网络/存储标识。
   - 常用于数据库、队列等。
5. DaemonSet
   - 确保每个节点上运行一个 Pod 副本。
   - 用于日志收集、监控代理等。
6. Job & CronJob
   - Job：一次性任务，执行完成后停止。
   - CronJob：定时任务，类似 Linux 的 cron。

------

## **三、服务发现与负载均衡**

1. Service
   - 抽象 Pod 的访问入口，实现负载均衡。
   - 类型：ClusterIP、NodePort、LoadBalancer、ExternalName。
2. Ingress
   - HTTP/HTTPS 路由入口，支持域名、路径路由、TLS。
   - 常配合 Ingress Controller 使用。

------

## **四、存储管理**

1. Volume
   - Pod 内部的存储卷，生命周期与 Pod 相同。
2. PersistentVolume (PV)
   - 集群中独立于 Pod 的存储资源，管理员创建。
3. PersistentVolumeClaim (PVC)
   - Pod 对存储的请求，绑定 PV。
4. StorageClass
   - 动态供给 PV 的模板和策略。

------

## **五、配置与密钥管理**

1. ConfigMap
   - 存储非敏感配置，供 Pod 使用。
2. Secret
   - 存储敏感信息（密码、token），Pod 可挂载或通过环境变量访问。

------

## **六、命名与访问管理**

1. Namespace
   - 多租户隔离，不同环境或团队可以用不同 Namespace。
2. RBAC（Role-Based Access Control）
   - 通过 Role/ClusterRole 和 RoleBinding/ClusterRoleBinding 管理权限。

------

## **七、调度与集群自愈**

1. NodeSelector / NodeAffinity / Taints & Tolerations
   - 控制 Pod 调度到指定节点或节点类型。
2. Horizontal Pod Autoscaler (HPA)
   - 根据 CPU/内存或自定义指标自动扩缩 Pod。
3. PodDisruptionBudget (PDB)
   - 控制 Pod 在节点维护或缩容时的最小可用数。