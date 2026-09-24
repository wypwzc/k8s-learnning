# Kubernetes 架构与核心组件（面试/日常脱口而出版）

------

## 一、Control Plane（Master）组件

### 1. API Server（kube-apiserver）

**一句话**：**集群的唯一入口，所有操作的"前台接待"**。

- 暴露 REST API，接收 `kubectl`、其他组件、外部系统的所有请求
- 做**认证（Authentication）**、**鉴权（Authorization）**、**准入控制（Admission Control）**
- 是**唯一**能直接读写 etcd 的组件，其他组件必须通过它
- 无状态，可水平扩展（多副本 + 负载均衡）

### 2. etcd

**一句话**：**集群的"数据库"，存储所有配置和状态**。

- 分布式键值存储（基于 Raft 协议），保证一致性
- 存储：Pod、Service、Deployment、Secret、ConfigMap 等所有 K8s 资源
- 只与 API Server 通信，其他组件不直接访问
- **性能敏感**：etcd 慢 = 整个集群慢

### 3. Scheduler（kube-scheduler）

**一句话**：**"调度员"，决定 Pod 放到哪个 Node 上**。

- 监听 API Server 中未分配 Node 的 Pod（`spec.nodeName` 为空）
- 经过**预选（Predicates）**和**优选（Priorities）**两个阶段：
  - **预选**：过滤掉不满足条件的节点（资源不足、污点排斥、亲和性不满足等）
  - **优选**：对剩余节点打分，选最高分（如资源利用率均衡、亲和性权重等）
- 只负责**决策**，不负责**执行**（执行是 kubelet 的事）

### 4. Controller Manager（kube-controller-manager）

#### 控制器管理器

**一句话**：**"管家"，确保集群实际状态 = 期望状态**。

- 由多个控制器组成，每个控制器是一个独立的控制循环（Control Loop）：
  - **Node Controller**：节点宕机后，驱逐该节点上的 Pod，标记 NotReady
  - **Replication Controller**：确保 ReplicaSet/Deployment 的副本数正确
  - **Endpoints Controller**：维护 Service 和 Pod 的映射关系（生成 Endpoints）
  - **Service Account & Token Controller**：为命名空间创建默认 SA 和 Token
  - **Job/CronJob Controller**：管理批处理任务生命周期
- 持续 Watch API Server，发现偏差后发起调和（Reconcile）

------

## 二、Node（Worker）组件

### 1. kubelet

**一句话**：**Node 上的"代理"，负责 Pod 的生命周期管理**。

- 向 API Server 注册该 Node（发送 Node 状态、资源容量）
- 接收 API Server 下发的 PodSpec，管理容器的**创建、启停、健康检查**
- 调用 CRI（Container Runtime Interface）与容器运行时交互
- 定期向 API Server 上报 Pod 状态（Status）
- **如果 kubelet 挂了，该 Node 上的 Pod 不会被重新调度，但已运行的容器可能继续跑**

### 2. kube-proxy

**一句话**：**负责 Service 的网络转发和负载均衡**。

- 监听 API Server 中 Service 和 Endpoints 的变化
- 实现 Service 的虚拟 IP（ClusterIP）到后端 Pod IP 的映射
- 三种代理模式：
  - **iptables**（默认）：为每个 Service 创建 iptables 规则，NAT 转发
  - **ipvs**：内核态负载均衡，性能更高，支持更多算法
  - **userspace**（已废弃）：早期用户态转发，性能差
- 只负责**四层（TCP/UDP）**转发，**七层路由**由 Ingress Controller 负责

### 3. Container Runtime

**一句话**：**真正运行容器的"引擎"**。

- 早期：Docker（通过 dockershim 对接，K8s 1.24 已移除）
- 现在主流：**containerd**、CRI-O
- 通过 CRI（gRPC 接口）与 kubelet 通信
- 负责：镜像拉取、容器创建、资源隔离（cgroups）、网络配置（CNI）

------

## 三、Pod 从创建到运行的完整流程

```plain
用户执行 kubectl apply -f pod.yaml
        ↓
┌─────────────────────────────────────────────────────────────┐
│  1. kubectl  →  API Server                                    │
│     - 客户端认证（证书/Token）                                 │
│     - 请求经过 AuthN → AuthZ → Admission Webhook               │
│     - 写入 etcd（Pod 资源对象，status.phase = Pending）         │
└─────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────┐
│  2. etcd 确认写入成功，返回给 API Server                       │
│     - API Server 通过 Watch 机制通知所有监听者                  │
└─────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────┐
│  3. Scheduler 监听到新 Pod（nodeName 为空）                    │
│     - 预选：过滤不符合的节点                                   │
│     - 优选：打分选最佳节点                                     │
│     - 将 nodeName 写入 Pod 的 spec，通过 API Server 更新 etcd   │
│     - Pod status.phase = Pending（已调度）                      │
└─────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────┐
│  4. 目标 Node 的 kubelet 监听到 Pod 被分配到自己                 │
│     - 调用 CRI（containerd）创建 Pause 容器（infra 容器）       │
│     - 调用 CNI 插件配置 Pod 网络（分配 IP、设置 veth pair）      │
│     - 调用 CRI 拉取业务镜像（如 nginx:latest）                  │
│     - 启动业务容器，设置 cgroups 资源限制                      │
│     - 执行 PostStart Hook、健康检查（Liveness/Readiness）      │
└─────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────┐
│  5. kubelet 持续上报状态给 API Server → etcd                   │
│     - 容器启动成功：Pod status.phase = Running                 │
│     - Readiness 探针通过：Pod 加入 Service Endpoints           │
│     - 用户可通过 kubectl get pod 看到 Running 状态              │
└─────────────────────────────────────────────────────────────┘
```

------

## 四、一句话速记（面试用）

| 组件                   | 速记口诀                   |
| :--------------------- | :------------------------- |
| **API Server**         | 唯一入口，接待所有请求     |
| **etcd**               | 集群数据库，存所有状态     |
| **Scheduler**          | 调度员，Pod 分配到哪里     |
| **Controller Manager** | 管家，发现偏差就纠正       |
| **kubelet**            | 节点代理，真正干活起容器   |
| **kube-proxy**         | 网络代理，Service 转 Pod   |
| **Container Runtime**  | 容器引擎，containerd/CRI-O |

------

## 五、高频追问

**Q：如果 Scheduler 挂了，已运行的 Pod 会受影响吗？**

> 不会。已运行的 Pod 不受影响，但新创建的 Pod 会一直处于 Pending 状态，无法调度。

**Q：kubelet 和 API Server 通信断了会怎样？**

> Node 状态变为 NotReady（默认 40 秒未上报）。5 分钟后（默认 `pod-eviction-timeout`），Controller Manager 会驱逐该节点上的 Pod，在其他节点重建。

**Q：为什么要有 Pause（infra）容器？**

> 1. 作为 Pod 的**根容器**，持有 Pod 的 Network Namespace 和 IPC Namespace
> 2. 业务容器通过 `container://pause` 共享网络栈
> 3. 业务容器崩溃重启时，Pod IP 不变（因为 Pause 容器一直活着）

