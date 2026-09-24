## 一、核心概念：用租房来理解

| K8s 概念 | 租房类比              | 一句话解释                                          |
| :------- | :-------------------- | :-------------------------------------------------- |
| **PV**   | 房东的房子            | 集群里**真实存在的存储资源**（一块盘、一个NFS目录） |
| **PVC**  | 租客的租房需求        | Pod 说"我要 5Gi 的存储"，这是**用户的申请**         |
| **SC**   | 房产中介/自动租房平台 | 收到 PVC 后，**自动去创建 PV**，不用管理员手动建    |
| **Pod**  | 租客本人              | 最终**住进去使用**存储的人                          |

### 两种供给模式

```plain
【静态供给】管理员手动创建 PV
管理员创建 PV ──→ 用户创建 PVC ──→ Pod 使用
   (房东有房)      (租客挑房)       (入住)

【动态供给】StorageClass 自动创建 PV
用户创建 PVC ──→ SC 自动创建 PV ──→ Pod 使用
   (我要租房)      (中介自动找房)     (入住)
```

------

## 二、每个概念拆开讲

### 1. PV（PersistentVolume）

就是**真实的存储**，由集群管理员预先准备好。它描述了：

- 存储类型（NFS、Ceph、云盘 EBS 等）
- 容量（10Gi）
- 访问模式（单节点读写/多节点只读/多节点读写）
- 回收策略（删除/保留）

### 2. PVC（PersistentVolumeClaim）

就是**用户的申请单**，Pod 不直接绑定 PV，而是通过 PVC 来"申请"：

- 我要 5Gi
- 我要 ReadWriteOnce 模式
- 我要绑到 NFS 上

K8s 会自动找一个**满足条件且未被绑定**的 PV 挂上来。

### 3. StorageClass（SC）

**动态供给的核心**。你提前定义好一个 SC（比如叫 `nfs-client`），里面写明：

- 用哪个 provisioner（自动创建 PV 的程序）
- 用什么后端（NFS 服务器地址、路径模板）
- 参数（归档策略等）

以后用户只需要写 PVC 并指定 `storageClassName: nfs-client`，PV 就会**自动冒出来**。

------

## 三、PV 与 PVC 生命周期详解（重点）

理解生命周期是面试和排障的核心。一个完整的存储生命周期分为四个阶段：**构建（Provisioning）→ 绑定（Binding）→ 使用（Using）→ 回收（Reclaiming）**。

### 3.1 生命周期总览

```plain
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Provisioning│ ──→ │   Binding   │ ──→ │   Using     │ ──→ │  Reclaiming │
│   (构建)     │     │   (绑定)    │     │   (使用)     │     │   (回收)     │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
```

### 3.2 阶段一：构建（Provisioning）

构建是指**将外部存储资源注册为 K8s 中的 PV 对象**的过程，分为两种模式：

#### ① 静态构建（Static Provisioning）

由集群管理员**手动**预先创建 PV，对应文档第四部分的静态供给。

- **特点**：管理员需要提前知道后端存储的详细信息（IP、路径、容量等）
- **适用场景**：存量数据迁移、特殊存储需求、需要精确控制存储位置时
- **状态流转**：`Available`（创建后等待绑定）→ `Bound`（绑定后）→ `Released`（PVC 删除后）

yaml

```yaml
# 静态构建示例：管理员手动创建 PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-static-nfs-001
  labels:
    type: nfs
    env: production
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""          # 空字符串表示不绑定任何 SC，防止被动态绑定
  mountOptions:
    - hard
    - nolock
    - nfsvers=4.1
  nfs:
    server: 192.168.1.100
    path: /data/static/nginx
```

#### ② 动态构建（Dynamic Provisioning）

用户提交 PVC 后，由 **StorageClass + Provisioner** 自动创建 PV，对应文档第五部分的动态供给。

- **特点**：用户无需关心后端细节，按需自动创建
- **适用场景**：云原生环境、DevOps 自助服务、大规模集群
- **状态流转**：PVC 创建后先进入 `Pending` → Provisioner 自动创建 PV → PV 和 PVC 同时进入 `Bound`

yaml

```yaml
# 动态构建触发器：用户只需提交 PVC，无需手动创建 PV
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dynamic-trigger
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-client   # 指定 SC 后，Provisioner 会自动创建 PV
  resources:
    requests:
      storage: 5Gi
```

### 3.3 阶段二：绑定（Binding）

绑定是 K8s 控制平面将 **PV 与 PVC 进行一对一匹配**的过程。

#### 绑定机制

plain

```plain
用户创建 PVC
    │
    ▼
┌─────────────────┐
│ PVC 进入 Pending │  ← 如果是动态供给，此时会触发 Provisioner 创建 PV
│   (等待匹配)     │
└─────────────────┘
    │
    ▼
K8s 调度器寻找满足条件的 PV：
  1. 容量 ≥ PVC 请求
  2. accessModes 匹配
  3. storageClassName 一致（PVC 未指定则匹配默认 SC）
  4. PV 状态为 Available
    │
    ▼
PV 与 PVC 双向绑定：
  • PV.spec.claimRef 指向 PVC
  • PVC.spec.volumeName 指向 PV
    │
    ▼
┌─────────────────┐
│  PVC 变为 Bound  │
│  PV 变为 Bound   │
└─────────────────┘
```

#### 绑定规则

| 场景                    | 行为                                                   |
| :---------------------- | :----------------------------------------------------- |
| PVC 指定 `volumeName`   | 强制绑定到该 PV，无论容量/模式是否匹配（需管理员确认） |
| PVC 未指定 `volumeName` | 自动匹配最优 PV（容量最小满足、accessModes 匹配）      |
| 无可用 PV 且无匹配 SC   | PVC 持续 `Pending`                                     |
| `storageClassName: ""`  | 只匹配未设置 SC 的 PV（静态专用）                      |
| 多个 PVC 竞争一个 PV    | 先到先得，其余 PVC 继续 Pending                        |

#### 绑定延迟模式：`volumeBindingMode`

yaml

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client-delayed
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
volumeBindingMode: WaitForFirstConsumer   # 延迟绑定，等 Pod 调度后再绑定
```

- **`Immediate`**（默认）：PVC 创建后立即绑定，不管 Pod 调度到哪个节点。可能导致 Pod 被调度到没有该 PV 的节点上（如云盘只能在特定可用区）。
- **`WaitForFirstConsumer`**：等 Pod 被调度后，再根据 Pod 所在节点的拓扑约束选择或创建 PV。**有状态服务（StatefulSet）强烈推荐**。

### 3.4 阶段三：使用（Using）

Pod 通过 `volumes.persistentVolumeClaim` 引用 PVC，K8s 将 PV 对应的存储挂载到容器内。

#### 使用流程

```plain
Pod 创建
  │
  ▼
Kubelet 发现 Pod 使用了 PVC
  │
  ▼
检查 PVC 是否已 Bound（未 Bound 则 Pod 处于 Pending）
  │
  ▼
Kubelet 调用存储插件（in-tree 或 CSI）执行挂载：
  • NFS: mount -t nfs 192.168.1.100:/data/k8s /var/lib/kubelet/...
  • EBS: 调用云 API AttachVolume → 格式化 → 挂载
  │
  ▼
存储挂载到宿主机目录，再通过 bind mount 进入容器
  │
  ▼
Pod 进入 Running，容器内应用正常读写
```

#### 使用中的状态

| 资源 | 状态      | 含义                                  |
| :--- | :-------- | :------------------------------------ |
| PVC  | `Bound`   | 已绑定，正在被 Pod 使用               |
| PV   | `Bound`   | 已绑定到某个 PVC，不可被其他 PVC 使用 |
| Pod  | `Running` | 挂载成功，容器启动                    |

**注意**：一个 PV 只能同时绑定一个 PVC；一个 PVC 可以被多个 Pod 同时挂载（取决于 accessModes，如 RWX）。

### 3.5 阶段四：回收（Reclaiming）

当 PVC 被删除后，PV 进入释放阶段，根据 `persistentVolumeReclaimPolicy` 决定数据命运。

#### 回收策略详解

| 策略        | 行为                                                         | 数据是否保留 | PV 是否可复用                                | 适用场景                       |
| :---------- | :----------------------------------------------------------- | :----------- | :------------------------------------------- | :----------------------------- |
| **Retain**  | PVC 删除后，PV 变为 `Released`，数据保留在存储后端，PV 不可自动复用 | ✅ 保留       | ❌ 需管理员手动清理 `claimRef` 后才能重新绑定 | **生产环境核心数据**           |
| **Delete**  | PVC 删除后，PV 和数据一起被删除（动态供给时，Provisioner 会清理后端存储） | ❌ 删除       | ✅ 自动删除                                   | 临时数据、开发测试环境         |
| **Recycle** | PVC 删除后，PV 数据被清空（执行 `rm -rf`），PV 变回 `Available` | ❌ 清空       | ✅ 自动清空后可复用                           | **已废弃（K8s 1.15+ 不推荐）** |

#### Retain 策略下手动复用 PV

当使用 `Retain` 策略的 PVC 被删除后，PV 状态为 `Released`，需要管理员手动清理才能重新绑定：

bash

```bash
# 1. 查看 PV 状态
kubectl get pv pv-static-nfs-001
# STATUS: Released

# 2. 编辑 PV，清空 claimRef 字段（相当于解除"已出租"标记）
kubectl edit pv pv-static-nfs-001
# 删除以下部分：
# claimRef:
#   apiVersion: v1
#   kind: PersistentVolumeClaim
#   name: pvc-old-name
#   namespace: default
#   uid: xxxxxxxx

# 3. PV 状态恢复为 Available，可被新的 PVC 绑定
kubectl get pv pv-static-nfs-001
# STATUS: Available
```

#### 动态供给下的 Delete + 归档

动态供给中，`Delete` 策略可以配合参数实现"软删除"：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
parameters:
  archiveOnDelete: "true"    # 删除 PVC 时，目录重命名为 archived-pvc-xxx，而非 rm -rf
reclaimPolicy: Delete
```

### 3.6 PV 与 PVC 完整 YAML 配置参考

#### PV 完整配置清单

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-complete-example
  labels:
    app: nginx
    tier: frontend
  annotations:
    pv.kubernetes.io/bound-by-controller: "yes"
spec:
  # 容量
  capacity:
    storage: 50Gi
  
  # 访问模式（RWO / ROX / RWX）
  accessModes:
    - ReadWriteMany
  
  # 回收策略（Retain / Delete / Recycle）
  persistentVolumeReclaimPolicy: Retain
  
  # 存储类（空字符串表示不绑定 SC）
  storageClassName: nfs-manual
  
  # 卷模式（Filesystem 是默认，Block 用于裸块设备）
  volumeMode: Filesystem
  
  # 挂载选项
  mountOptions:
    - hard
    - nolock
    - nfsvers=4.1
  
  # 节点亲和性（限制哪些节点可以访问此 PV，本地存储常用）
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - node-01
  
  # 具体存储后端配置
  nfs:
    server: 192.168.1.100
    path: /data/k8s/pv-complete
```

#### PVC 完整配置清单

yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-complete-example
  namespace: default
  labels:
    app: nginx
spec:
  # 访问模式（必须与目标 PV 匹配）
  accessModes:
    - ReadWriteMany
  
  # 资源请求
  resources:
    requests:
      storage: 50Gi      # 请求容量
    # limits:             # PVC 通常不设 limits，由 PV 决定上限
    #   storage: 100Gi
  
  # 指定存储类（动态供给必需；静态供给可选，但建议一致）
  storageClassName: nfs-manual
  
  # 强制绑定到指定 PV（可选，用于精确控制）
  volumeName: pv-complete-example
  
  # 选择器（通过标签匹配 PV，不常用）
  selector:
    matchLabels:
      app: nginx
      tier: frontend
  
  # 卷模式
  volumeMode: Filesystem
```

#### Pod 挂载 PVC 完整配置

yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-pvc
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: persistent-storage
      mountPath: /usr/share/nginx/html
      readOnly: false        # 是否只读挂载
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: pvc-complete-example
      readOnly: false        # 是否强制只读
```

### 3.7 生命周期状态速查

```plain
PV 状态机：
  Available ──(绑定)──→ Bound ──(PVC删除)──→ Released ──(管理员清理claimRef)──→ Available
       ↑                                                            │
       └──────────────────(动态供给新建)──────────────────────────────┘

PVC 状态机：
  Pending ──(找到匹配PV)──→ Bound ──(被Pod使用)──→ Lost (PV 异常丢失时)
    │
    └──(无匹配PV/SC)──→ 持续 Pending
```

表格



| 状态        | 含义                                |
| :---------- | :---------------------------------- |
| `Available` | PV 可用，尚未绑定任何 PVC           |
| `Bound`     | PV 已绑定到 PVC，正在被使用         |
| `Released`  | PVC 已删除，PV 已释放，等待回收处理 |
| `Failed`    | PV 自动回收失败（极少见）           |
| `Pending`   | PVC 已创建，尚未找到匹配的 PV       |
| `Lost`      | PVC 绑定的 PV 丢失或不可用          |

------

## 四、实战：NFS 静态供给（最基础，必会）

假设你有一台 NFS 服务器 `192.168.1.100`，共享了 `/data/k8s`。

### Step 1：管理员创建 PV（手动）

yaml

```yaml
# pv-nfs.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-001
spec:
  capacity:
    storage: 5Gi                    # 容量
  accessModes:
    - ReadWriteMany                 # 多节点读写（NFS 支持）
  persistentVolumeReclaimPolicy: Retain  # 回收策略：保留数据
  nfs:
    server: 192.168.1.100           # NFS 服务器
    path: /data/k8s                 # 共享目录
```

bash

```bash
kubectl apply -f pv-nfs.yaml
kubectl get pv
# STATUS 应该是 Available（可用）
```

### Step 2：用户创建 PVC（申请）

yaml

yaml

```yaml
# pvc-nfs.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs-001
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi                  # 申请 5Gi
  volumeName: pv-nfs-001            # 指定绑定哪个 PV（可选，不指定会自动匹配）
```

bash

bash

```bash
kubectl apply -f pvc-nfs.yaml
kubectl get pvc
# STATUS 应该是 Bound（已绑定）
kubectl get pv
# STATUS 会变成 Bound
```

### Step 3：Pod 使用 PVC

yaml

yaml

```yaml
# pod-use-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html   # 容器内挂载点
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-nfs-001              # 引用上面的 PVC
```

bash

bash

```bash
kubectl apply -f pod-use-pvc.yaml
kubectl exec -it web-pod -- sh
# 在容器里往 /usr/share/nginx/html 写文件
# 去 NFS 服务器 /data/k8s 看，文件已经同步过去了
```

------

## 五、实战：StorageClass + NFS 动态供给（进阶，面试加分）

“动态供给的核心是 **StorageClass** 与 **Provisioner** 的配合。用户只需提交 PVC，K8s 会通过 StorageClass 找到对应的 Provisioner（执行器）。Provisioner 监听到 PVC 后，自动在后台存储（如 NFS）中划拨空间、创建子目录，并生成对应的 PV 与 PVC 进行绑定，全程对应用开发者透明。”

静态供给的问题是：每来一个用户申请，管理员都要手动建 PV，太麻烦。

动态供给就是：**用户只提 PVC，PV 自动创建**。

### 架构原理

plain

```plain
用户创建 PVC (指定 storageClassName: nfs-client)
        ↓
nfs-subdir-external-provisioner 监听到 PVC
        ↓
在 NFS 服务器上自动创建子目录 /data/k8s/pvc-xxx
        ↓
自动创建 PV 并绑定
        ↓
Pod 正常使用
```

### Step 1: 部署 RBAC 与 Provisioner（执行器）

在 K8s 中，Provisioner 是一个独立运行的 Pod，它需要监听 PVC 事件、创建 PV 资源，这就必须要有相应的 RBAC 权限。

**1. 创建 `rbac.yaml`** 这个文件定义了 ServiceAccount，并赋予它操作 PV/PVC 以及 Endpoints 的权限：



```plain
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  namespace: default
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: default
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: Role
    name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

**2. 创建 `provisioner.yaml`** 部署实际的 Provisioner 载体（注意修改 NFS Server IP 和路径）：

yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          # 推荐使用 v4 版本，兼容较高版本的 K8s
          image: registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 192.168.1.100  # [修改处] 你的 NFS 服务器 IP
            - name: NFS_PATH
              value: /data/k8s      # [修改处] 你的 NFS 共享目录
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.1.100   # [修改处] 必须和上面环境变量一致
            path: /data/k8s         # [修改处] 必须和上面环境变量一致
```

### Step 2: 创建 StorageClass

告诉 K8s 哪种类型的存储要找哪个 Provisioner：

**创建 `storageclass.yaml`**``

YAML

plain

```plain
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner  # 必须与 Provisioner 环境变量中的 PROVISIONER_NAME 保持一致
parameters:
  archiveOnDelete: "true"  # PVC 删除时，数据目录改名为 archived-xxx，而非彻底清空（防数据误删）
reclaimPolicy: Delete      # 注意：只有在 Delete 模式下，archiveOnDelete 参数才会生效
volumeBindingMode: Immediate
```

### Step 3: 连通性与功能测试 (PVC + Pod)

把 PVC 和 Pod 写在一个文件中进行闭环测试：

**创建 `test-pvc-pod.yaml`**

yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dynamic-test
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-client  # 关键：指定上面的 SC 名字
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-nfs-test
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-dynamic-test  # 引用上面的 PVC
```

一键部署：`kubectl apply -f rbac.yaml -f provisioner.yaml -f storageclass.yaml -f test-pvc-pod.yaml`

### 运维排坑：PVC 处于 Pending 状态的常见原因（重点）

在实战中，如果 `kubectl get pvc` 看到状态一直卡在 **Pending**，可以按照以下思路排查（面试必考的 Troubleshooting）：

1. **StorageClass 名字拼写错误**
   - **现象**：PVC 的 `storageClassName` 和系统里存在的 StorageClass 不一致。
   - **排查**：`kubectl describe pvc <pvc-name>`，如果看到 `storageclass.storage.k8s.io "xxx" not found` 就是这个问题。
2. **Provisioner Pod 未正常运行**
   - **现象**：nfs-client-provisioner 所在的 Pod 挂了（CrashLoopBackOff 或者 ImagePullBackOff）。
   - **排查**：`kubectl get pods` 检查 provisioner 状态。如果 Pod 起不来，多半是 NFS 服务端没配置对（网络不通、或者服务端的 `/etc/exports` 没加上你的 K8s 节点 IP）。
3. **RBAC 权限缺失**
   - **现象**：Provisioner Pod 跑着，但是日志里疯狂报错 `forbidden: User "system:serviceaccount:default:xxx" cannot list resource "persistentvolumes"...`。
   - **排查**：`kubectl logs -l app=nfs-client-provisioner`。仔细核对上面的 `rbac.yaml` 是否漏建或者 ServiceAccount 名字没对上。
4. **NFS 服务端权限问题（`root_squash` 坑）**
   - **现象**：Provisioner 报错 `mkdir: cannot create directory ... Permission denied`。K8s 中的 root 用户操作 NFS 时被服务端降级为了普通用户（nobody/nfsnobody）。
   - **排查/解决**：去 NFS 服务器上检查 `/etc/exports`，确保挂载参数里带了 `no_root_squash`，例如：`/data/k8s *(rw,sync,no_root_squash)`，修改后记得 `exportfs -arv` 生效。
5. **K8s 1.20+ 版本的 SelfLink 特性被废弃**
   - **现象**：如果你用的是老版本的 Provisioner 镜像（v3.x 及以下），在 K8s 1.20+ 版本上会报错 `unexpected error getting claim reference: selfLink was empty`。
   - **排查/解决**：这是由于 K8s 移除了 `RemoveSelfLink` 特性。最正规的解法是把 Provisioner 镜像升级到上面 YAML 中提供的 `v4.0.2` 或更高版本。

------

## 六、关键参数速查表（面试常问）



| 参数                  | 含义     | 常用值                                                       |
| :-------------------- | :------- | :----------------------------------------------------------- |
| **accessModes**       | 访问模式 | `ReadWriteOnce`(RWO，单节点读写)、`ReadOnlyMany`(ROX，多节点只读)、`ReadWriteMany`(RWX，多节点读写) |
| **reclaimPolicy**     | 回收策略 | `Retain`(保留数据，PV 不删)、`Delete`(删 PVC 时连 PV 和数据一起删)、`Recycle`(已废弃，清空数据) |
| **volumeBindingMode** | 绑定时机 | `Immediate`(立即绑定，不管 Pod 在哪)、`WaitForFirstConsumer`(等 Pod 调度后再绑定，配合拓扑约束) |
| **storageClassName**  | 指定 SC  | `nfs-client`、`standard`、`gp2`(AWS) 等                      |

### 一个面试高频考点

> **问：RWO、ROX、RWX 有什么区别？NFS 支持哪种？**
>
> - **RWO**：只能一个节点挂载读写（比如 AWS EBS、本地盘）
> - **ROX**：多个节点都能挂载，但只能读
> - **RWX**：多个节点都能挂载读写（**NFS 支持这个**，所以 NFS 适合多 Pod 共享存储）
>
> 云盘（EBS、阿里云盘）一般只支持 RWO，NFS、CephFS、GlusterFS 支持 RWX。

------

## 七、排障思路（实习生必须会）

### 场景 1：PVC 一直 Pending

```bash
kubectl describe pvc pvc-name
```

常见原因：

- **没有可用的 PV**：PV 容量不够、accessModes 不匹配、PV 已经被绑了
- **没有匹配的 StorageClass**：SC 名字写错，或者 provisioner 挂了
- **WaitForFirstConsumer**：Pod 没调度，PVC 不会绑定

### 场景 2：Pod 起不来，报 mount 失败

```bash
kubectl describe pod pod-name
```

常见原因：

- NFS 服务器不通（`showmount -e 192.168.1.100` 测一下）
- NFS 客户端没装（节点上需要 `nfs-utils`）
- 路径写错

### 场景 3：数据丢了

检查 PV 的 `persistentVolumeReclaimPolicy`：

- 如果是 `Delete`，删 PVC 时 PV 和数据会被清理
- 生产环境重要数据一定要用 `Retain`

------

## 八、云计算运维实习生面试：要掌握到什么程度？

### 必须掌握（100% 会问）

1. **概念清晰**：能说出 PV 是资源、PVC 是申请、SC 是自动创建，三者关系
2. **静态 vs 动态**：能讲清楚区别，什么时候用静态、什么时候用动态
3. **访问模式**：RWO、ROX、RWX 的区别，NFS 支持 RWX
4. **回收策略**：Retain 和 Delete 的区别，生产环境为什么用 Retain
5. **会写 YAML**：能独立写出 PV、PVC、Pod 挂载的 YAML
6. **NFS 基础**：知道 NFS 是网络文件系统，支持多节点共享，节点要装 `nfs-utils`

### 加分项（说出来能拉开差距）

1. **能讲动态供给原理**：provisioner 监听 PVC → 自动创建目录 → 自动创建 PV
2. **知道 WaitForFirstConsumer**：配合有状态服务、拓扑感知调度
3. **排障能力**：PVC Pending、mount 失败、数据丢失的排查思路
4. **了解其他后端**：除了 NFS，还知道 Ceph、云盘 EBS/SSD、hostPath 的适用场景
5. **StatefulSet 配套**：知道 StatefulSet 的 volumeClaimTemplates 配合 SC 自动为每个 Pod 分配独立存储

### 了解即可（不用深入）

- CSI 插件开发原理
- Ceph 的 CRUSH 算法
- 具体云厂商的存储 API

------

## 九、总结记忆口诀

> **PV 是房，PVC 是单，SC 是中介自动建。** **静态管理员手动配，动态用户只管提需求。** **NFS 支持多节点读写，云盘一般只能单节点。** **Retain 留数据，Delete 全清掉，生产环境要慎重。**

