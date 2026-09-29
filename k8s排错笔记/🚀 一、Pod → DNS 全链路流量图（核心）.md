k8s网络层

👉 **Flannel 解决的是 Pod-to-Pod 网络**
 👉 **kube-proxy 解决的是 Pod-to-Service 网络**
 👉 **CoreDNS 依赖 Service 网络**

#  一、Pod → DNS 全链路流量图（核心）

```
[ Pod ]
   |
   | ① DNS 请求 (nslookup kubernetes.default)
   v
[ Pod 内 /etc/resolv.conf ]
   |
   | nameserver = 10.96.0.10
   v
[ ClusterIP: kube-dns Service ]
   |
   | ② kube-proxy (iptables / ipvs)
   v
[ Node Linux 内核转发 ]
   |
   | ③ flannel overlay (vxlan)
   v
[ CoreDNS Pod (10.244.x.x) ]
   |
   | ④ 查询 kube-apiserver / upstream DNS
   v
[ 返回结果 → Pod ]
```

------

# 二、4层断点模型（面试必杀）

你可以把 K8s 网络问题拆成 4 层：

------

## L1：Pod 内部 DNS

### 检查：

```
cat /etc/resolv.conf
```

### 正常：

```
nameserver 10.96.0.10
```

### ❌ 如果错：

→ CoreDNS 没注入 / Pod 配置问题

------

## 🟡 L2：Service 层（kube-proxy）

### 检查：

```
kubectl get svc -n kube-system kube-dns
```

### 检查 iptables：

```
iptables -t nat -L -n | grep 10.96.0.10
```

### ❌ 如果没规则：

→ kube-proxy 没生效

------

## 🟠 L3：Pod 网络（Flannel / CNI）

### 测试：

```
ping 10.244.x.x
```

### ❌ 不通：

→ Flannel / VXLAN 问题

------

## 🔴 L4：Node 内核转发

### 必查：

```
sysctl net.ipv4.ip_forward
iptables -L FORWARD
ip route
```

### ❌ 问题：

→ Service CIDR 无法路由（你刚才就是这个）

------

# 🔥 三、你这次问题的真实断点（重点）

你这次其实是：

```
Pod DNS失败
   ↓
Service 10.96.0.10 不通
   ↓
ip route 没有 10.96.0.0/12
   ↓
kube-proxy 没正确建立转发路径
```

------

# 🧭 四、标准排障方法（面试直接说这个）

记住这一套就够了：

------

## ✅ Step 1：Pod 是否正常

```
kubectl get pods
```

------

## ✅ Step 2：DNS 是否配置

```
cat /etc/resolv.conf
```

------

## ✅ Step 3：Service 是否存在

```
kubectl get svc
```

------

## ✅ Step 4：Service IP 是否可达

```
ping 10.96.0.10
```

------

## ✅ Step 5：iptables / ipvs

```
iptables -t nat -L -n | grep KUBE
```

------

## ✅ Step 6：路由

```
ip route
```

------

## ✅ Step 7：CNI 网络

```
ping 10.244.x.x
```

------

# 🧠 五、记忆口诀（很重要）

👉 面试直接说这句：

```
Pod 不通 DNS：
先看 resolv.conf
再看 kube-dns Service
再看 kube-proxy iptables
再看 Pod 网络 flannel
最后看内核路由 sysctl
```

------

# 🚀 六、给你一个升级版理解（高手分界线）

K8s 网络其实就是三张表：

| 层           | 控制组件         |
| ------------ | ---------------- |
| Pod 网络     | Flannel / Calico |
| Service 网络 | kube-proxy       |
| DNS          | CoreDNS          |