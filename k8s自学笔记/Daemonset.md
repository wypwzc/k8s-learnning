# Kubernetes DaemonSet 全面教程

在 Kubernetes 中，`DaemonSet` 是一个非常核心的工作负载资源。

它的作用只有一句话：

> **保证每个 Node（节点）上都运行一个 Pod。**

也就是说：

- 集群新增 Node → 自动创建 Pod
- 删除 Node → Pod 自动消失
- 每台机器都部署同样的服务

这就是 DaemonSet。

------

# 一、为什么会有 DaemonSet？

先思考一个问题：

有些服务是不是必须每台机器都装？

例如：

| 场景          | 说明                 |
| ------------- | -------------------- |
| 日志收集      | 每台节点都要收集日志 |
| 网络插件      | 每台节点都要配置网络 |
| 监控 Agent    | 每台节点都要监控     |
| kube-proxy    | 每台节点都需要       |
| node-exporter | 每台节点都需要       |

这些都属于：

> “节点级服务”

这时候 Deployment 不适合。

因为：

Deployment：

- 只保证副本数
- 不保证每个节点都有

例如：

```
replicas: 3
```

可能：

```
node1: 2个Pod
node2: 1个Pod
node3: 0个Pod
```

但 DaemonSet：

```
node1: 1个
node2: 1个
node3: 1个
```

这就是区别。

------

# 二、DaemonSet 工作原理

## Deployment 的调度方式

Deployment：

```
Deployment
    ↓
ReplicaSet
    ↓
Scheduler
    ↓
Node
```

Scheduler 负责决定 Pod 去哪里。

------

## DaemonSet 的方式

DaemonSet：

```
DaemonSet Controller
       ↓
直接给每个Node创建Pod
```

它会：

```
遍历所有Node
    ↓
每个Node创建一个Pod
```

所以：

> DaemonSet 本质是“按节点创建 Pod”。

------

# 三、最经典的 DaemonSet

K8s 集群里你一定见过：

```
kubectl get pods -A
```

你会看到：

```
kube-flannel-ds
kube-proxy
calico-node
```

这些几乎都是 DaemonSet。

因为：

- 每台机器都需要
- 必须跟节点绑定

------

# 四、创建第一个 DaemonSet

------

## 1. 编写 YAML

```
apiVersion: apps/v1
kind: DaemonSet

metadata:
  name: nginx-ds

spec:
  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx

    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

------

# 五、部署

```
kubectl apply -f ds.yaml
```

查看：

```
kubectl get ds
```

你会看到：

```
NAME       DESIRED   CURRENT   READY
nginx-ds   3         3         3
```

含义：

| 字段    | 含义       |
| ------- | ---------- |
| DESIRED | 期望节点数 |
| CURRENT | 当前运行数 |
| READY   | 就绪数     |

如果你有 3 个 Node：

```
master
node1
node2
```

那么：

```
每个节点1个Pod
```

------

# 六、查看 Pod 分布

```
kubectl get pods -o wide
```

例如：

```
NAME             NODE
nginx-ds-aaa     master
nginx-ds-bbb     node1
nginx-ds-ccc     node2
```

你会发现：

> 每个节点一个。

------

# 七、DaemonSet 自动扩容原理

假设：

现在有：

```
master
node1
node2
```

后来新增：

```
node3
```

加入集群：

```
kubeadm join ...
```

DaemonSet Controller 会发现：

```
node3 没有Pod
```

于是：

```
自动创建一个新的Pod
```

这就是：

> “节点驱动型扩容”

而不是：

> “副本驱动型扩容”

这是 DaemonSet 的核心。

------

# 八、DaemonSet 与 Deployment 的区别

| 对比      | Deployment | DaemonSet            |
| --------- | ---------- | -------------------- |
| 控制方式  | 副本数     | 节点数               |
| 调度目标  | 任意节点   | 每个节点             |
| 自动扩容  | replicas   | Node数量             |
| 常见用途  | Web服务    | 节点Agent            |
| Scheduler | 会调度     | 基本绕过普通调度逻辑 |

------

# 九、限制只在某些节点运行

有时候：

你不想每台机器运行。

比如：

```
只在 worker 节点运行
```

------

## 方法1：nodeSelector

给节点打标签：

```
kubectl label node node1 disk=ssd
```

查看：

```
kubectl get nodes --show-labels
```

------

## YAML

```
spec:
  template:
    spec:
      nodeSelector:
        disk: ssd
```

意思：

```
只在有 disk=ssd 标签的节点运行
```

------

# 十、污点容忍（非常重要）

很多时候：

master 有污点：

```
NoSchedule
```

所以：

DaemonSet 默认可能不上 master。

查看污点：

```
kubectl describe node master
```

你会看到：

```
Taints:
node-role.kubernetes.io/control-plane:NoSchedule
```

------

## tolerations

如果想运行到 master：

```
spec:
  template:
    spec:
      tolerations:
      - operator: Exists
```

这表示：

```
容忍所有污点
```

------

# 十一、生产最常见 DaemonSet

------

## 1. Flannel

网络插件：

```
每台机器都要配置网络
```

所以必须 DaemonSet。

------

## 2. Calico

也是网络组件。

------

## 3. kube-proxy

每个节点都维护 iptables/ipvs 规则。

所以：

```
必须每个节点都运行
```

------

## 4. node-exporter

Prometheus 节点监控。

每个 Node 都要暴露：

```
CPU
内存
磁盘
网络
```

------

## 5. Fluentd / Filebeat

日志采集。

每个节点收集：

```
/var/log/containers
```

------

# 十二、实战：部署 node-exporter

这是面试特别爱问的。

------

## YAML

```
apiVersion: apps/v1
kind: DaemonSet

metadata:
  name: node-exporter
  namespace: monitoring

spec:
  selector:
    matchLabels:
      app: node-exporter

  template:
    metadata:
      labels:
        app: node-exporter

    spec:
      hostNetwork: true
      hostPID: true

      containers:
      - name: node-exporter
        image: prom/node-exporter

        ports:
        - containerPort: 9100
```

------

# 十三、为什么 DaemonSet 常用 hostPath

因为：

节点级服务经常需要：

```
访问宿主机
```

例如：

日志：

```
/var/log
```

容器信息：

```
/var/lib/docker
```

系统信息：

```
/proc
/sys
```

所以：

```
volumes:
- hostPath:
```

特别常见。

------

# 十四、更新策略

DaemonSet 也支持滚动更新。

------

## RollingUpdate

默认：

```
updateStrategy:
  type: RollingUpdate
```

效果：

```
逐台机器更新
```

不会一次全挂。

------

## OnDelete

```
updateStrategy:
  type: OnDelete
```

只有：

```
kubectl delete pod
```

才会更新。

------

# 十五、面试高频问题

------

## 1. DaemonSet 和 Deployment 区别？

核心：

```
Deployment 按副本
DaemonSet 按节点
```

------

## 2. 为什么 kube-proxy 用 DaemonSet？

因为：

每个 Node 都必须维护：

```
iptables/ipvs
```

------

## 3. DaemonSet 如何调度？

DaemonSet Controller：

```
遍历Node
创建Pod
```

不是普通副本调度。

------

## 4. DaemonSet 会自动扩容吗？

会。

新增 Node：

```
自动创建 Pod
```

------

# 十六、企业真实场景（重点）

真正生产里：

DaemonSet 几乎都是：

| 类型 | 示例               |
| ---- | ------------------ |
| 网络 | Flannel / Calico   |
| 监控 | node-exporter      |
| 日志 | Fluentd / Filebeat |
| 安全 | Falco              |
| 存储 | Ceph Agent         |

也就是说：

> DaemonSet = Node级 Agent 部署器

这是它真正的定位。

------

# 十七、最推荐你的学习顺序

你现在已经有：

- 1 master
- 2 node

非常适合练习。

建议你立刻做：

------

## 第一阶段

### 练习：

部署 nginx DaemonSet

学会：

```
kubectl get ds
kubectl describe ds
kubectl get pods -o wide
```

------

## 第二阶段

练习：

新增一个 Node

观察：

```
Pod 自动创建
```

这是理解 DaemonSet 最关键一步。

------

## 第三阶段（非常重要）

部署：

- node-exporter
- fluentd

理解：

```
为什么必须每节点一个
```

------

# 十八、你必须记住的一句话

> Deployment 管应用
>  DaemonSet 管节点

这是 Kubernetes 里最经典的一句话。