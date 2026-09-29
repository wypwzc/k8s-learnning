# 为什么需要 Service？

你已经知道：

- Pod 会运行应用
- Pod 有 IP
- Pod 可以通信

但问题来了：

Pod 是会“漂移”的。

例如：

```
Deployment 管理 3 个 nginx Pod：

10.244.0.2
10.244.1.5
10.244.2.8
```

如果某个 Pod 挂了：

```
10.244.1.5 -> 消失
新 Pod -> 10.244.3.9
```

所以：

❌ Pod IP 不稳定
 ❌ 不能让用户直接访问 Pod IP

于是：

K8s 发明了：

# Service

本质：

> 给一组 Pod 提供一个“固定入口”

你可以理解为：

```
Service = Kubernetes 内置负载均衡器
```

------

# Service 最核心原理

Service 并不是一个真正的进程。

它本质依赖：

```
kube-proxy
+
iptables/ipvs
```

实现流量转发。

------

# 整体架构图

```
用户
 ↓
Service(IP固定)
 ↓
kube-proxy
 ↓
iptables/ipvs规则
 ↓
后端多个 Pod
```

------

# 一、ClusterIP（最重要）

默认 Service 类型。

------

## 创建 Service

Deployment：

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx

spec:
  replicas: 3

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
        image: nginx
```

------

Service：

```
apiVersion: v1
kind: Service

metadata:
  name: nginx-service

spec:
  selector:
    app: nginx

  ports:
  - port: 80
    targetPort: 80
```

------

# selector 是核心

```
selector:
  app: nginx
```

意思：

```
把所有 app=nginx 的 Pod
加入负载均衡池
```

------

# Service 创建后会发生什么？

你执行：

```
kubectl apply -f service.yaml
```

K8s：

## 第一步：

给 Service 分配虚拟 IP

例如：

```
10.96.0.10
```

这叫：

# ClusterIP

集群内部 IP。

------

## 第二步：

kube-proxy 监听到 Service

每个 Node 上都有：

```
kube-proxy
```

它负责：

```
监听 Service/Endpoint 变化
生成 iptables/ipvs 规则
```

------

# 三、Endpoint（非常重要）

Service 后面到底有哪些 Pod？

K8s 会自动生成：

```
kubectl get endpoints
```

例如：

```
nginx-service
10.244.0.2:80
10.244.1.5:80
10.244.2.8:80
```

意思：

```
Service 后端真实 Pod
```

### Service 和 Endpoint 的关系

```
Service
只是入口
```

真正转发目标：

```
Endpoint
```

------

### 整个流程（必须理解）

```
客户端
 ↓
ClusterIP(Service)
 ↓
kube-proxy
 ↓
读取 Endpoint
 ↓
iptables/ipvs
 ↓
真实 Pod
```

------

# 四、iptables 转发原理（核心）

假设：

```
Service IP:
10.96.0.10:80
```

后端 Pod：

```
10.244.0.2:80
10.244.1.5:80
10.244.2.8:80
```

kube-proxy 会生成 iptables 规则：

```
访问 10.96.0.10:80
↓
随机转发到某个 Pod
```

例如：

```
客户端
 ↓
10.96.0.10
 ↓
iptables DNAT
 ↓
10.244.1.5
```

这里本质是：

# NAT 转发

你之前学 Docker 网络时已经接触过。

------

# 五、负载均衡怎么实现？

iptables 会：

```
随机选择 Pod
```

实现：

```
轮询负载均衡
```

效果：

```
请求1 -> Pod1
请求2 -> Pod2
请求3 -> Pod3
```

------

# 六、ClusterIP 特点

特点：

```
只能集群内部访问
```

例如：

Pod A：

```
curl 10.96.0.10
```

能访问。

但你 Windows 主机：

```
curl 10.96.0.10
```

访问不到。

因为：

```
ClusterIP 是虚拟内网IP
```

------

# 七、NodePort

如果想：

```
外部访问 K8s 服务
```

怎么办？

用：

# NodePort

------

## NodePort 原理

K8s：

```
在每个 Node 开一个端口
```

例如：

```
30080
```

然后：

```
NodeIP:30080
↓
Service
↓
Pod
```

------

# NodePort YAML

```
apiVersion: v1
kind: Service

metadata:
  name: nginx-nodeport

spec:
  type: NodePort

  selector:
    app: nginx

  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

------

# 访问方式

假设：

```
Node IP:
192.168.36.137
```

访问：

```
curl http://192.168.36.137:30080
```

即可访问 Pod。

------

# 流量路径

```
浏览器
 ↓
NodeIP:30080
 ↓
kube-proxy
 ↓
iptables/ipvs
 ↓
Service
 ↓
Pod
```

------

# 八、为什么每个 Node 都能访问？

即使 Pod 在 node2：

你访问：

```
master:30080
```

也能通。

因为：

```
kube-proxy 会跨节点转发
```

这是 K8s 网络强大的地方。

------

# 九、iptables 和 ipvs 区别（面试高频）

K8s Service 有两种模式：

------

## 1. iptables 模式（默认经典）

原理：

```
大量 iptables 规则
```

特点：

✅ 简单
 ✅ 稳定

缺点：

❌ Service 太多时性能下降

因为：

```
规则链太长
```

------

## 2. IPVS 模式（企业常用）

Linux 内核 LVS 技术。

特点：

✅ 高性能
 ✅ 大规模集群更强
 ✅ 真正负载均衡算法

支持：

```
rr 轮询
lc 最少连接
sh 源地址哈希
```

------

# 查看 kube-proxy 模式

```
kubectl logs -n kube-system kube-proxy-xxxxx
```

搜索：

```
Using iptables Proxier
```

或者：

```
Using ipvs Proxier
```

------

# 十、Service 的本质总结（必须背）

# ClusterIP 本质

```
一个虚拟IP
+
iptables/ipvs转发规则
```

------

# kube-proxy 本质

```
Service规则生成器
```

它不转发流量。

它只是：

```
写iptables/ipvs规则
```

真正转发的是：

```
Linux 内核
```

------

# NodePort 本质

```
在 Node 开端口
+
转发到 Service
```

------

# 十一、必须理解的网络流

# Pod 访问 Service

```
Pod
 ↓
ClusterIP
 ↓
iptables/ipvs
 ↓
目标 Pod
```

------

# 外部访问 NodePort

```
浏览器
 ↓
NodeIP:NodePort
 ↓
Service
 ↓
Pod
```

------

# 十二、你现在应该做的实验（非常重要）

建议你现在立刻做：

------

## 实验1：Deployment + ClusterIP

学会：

```
kubectl get svc
kubectl get endpoints
```

观察：

```
Service IP
Pod IP
```

------

## 实验2：NodePort

从 Windows：

```
curl NodeIP:30080
```

------

## 实验3：删除 Pod

观察：

```
Endpoint 自动变化
```

这是 Service 灵魂。

------

# 十三、你下一步该学什么？

你现在已经进入：

# K8s 网络核心阶段

下一步建议：

# Ingress

因为：

```
NodePort 太丑
```

企业不会大量用 NodePort。

------

# kube-proxy 深入

包括：

```
iptables
ipvs
conntrack
```

------

# CNI 网络插件

例如：

- Flannel
- Calico
- Cilium

这是：

```
Pod IP 为什么能跨节点通信
```

的核心。

------

# 最后一句（非常关键）

很多人学 Service：

只会写 YAML。

但真正重要的是理解：

```
Service 不是进程
不是容器
不是代理
```

而是：

# Linux 内核转发规则。