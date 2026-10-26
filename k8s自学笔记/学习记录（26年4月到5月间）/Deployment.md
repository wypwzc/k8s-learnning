# 一、Deployment 是干嘛的（一句话理解）

Deployment = **帮你管理 Pod 的“自动化控制器”**

它解决 4 个核心问题：

- 保证运行指定数量的 Pod（副本数）
- 支持滚动更新（不停机发布）
- 支持回滚（版本出问题恢复）
- 声明式管理（你写 YAML，它帮你实现）

👉 面试必答一句话：

> Deployment 用于管理无状态应用，负责副本控制、滚动更新和版本回滚

------

# 二、核心结构（一定要理解）

Deployment 并不是直接管理 Pod，它的结构是：

```
Deployment
   ↓
ReplicaSet
   ↓
Pod
```

👉 重点：

- Deployment 控制 ReplicaSet
- ReplicaSet 控制 Pod

你可以理解为：

> Deployment 是“老板”，ReplicaSet 是“经理”，Pod 是“员工”

------

# 三、最小可用示例（必须会写）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3   # 3个Pod
  selector:
    matchLabels:
      app: nginx
  template:     # Pod模板
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

------

# 四、实战操作（你要真的敲）

### 1️⃣ 创建 Deployment

```
kubectl apply -f nginx.yaml
```

### 2️⃣ 查看状态

```
kubectl get deploy
kubectl get rs
kubectl get pods
```

👉 你会看到：

- 1 个 Deployment
- 1 个 ReplicaSet
- 3 个 Pod

------

### 3️⃣ 扩缩容（超高频面试点）

```
kubectl scale deployment nginx-deployment --replicas=5
```

👉 作用：

- 从 3 个 Pod → 5 个 Pod

------

### 4️⃣ 滚动更新（最重要）

```
kubectl set image deployment/nginx-deployment nginx=nginx:1.26
```

👉 发生了什么：

- 新建一个 ReplicaSet
- 逐步替换旧 Pod
- 整个过程**不会宕机**

------

### 5️⃣ 回滚（救命操作）

```
kubectl rollout undo deployment/nginx-deployment
```

------

# 五、滚动更新原理（面试重点）

核心参数：

```
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # 最多多创建1个pod
    maxUnavailable: 1  # 最多允许1个pod不可用
```

👉 举例（3副本）：

- 最多同时存在 4 个 Pod
- 最少保证 2 个可用

------

# 六、常见面试坑（你必须知道）

### ❌ 1. selector 和 label 不一致

会导致：

> Deployment 管不到 Pod（严重问题）

------

### ❌ 2. 修改 label 会发生什么？

👉 会创建**全新的 ReplicaSet**（等于重建）

------

### ❌ 3. Deployment 适合什么场景？

✔ 无状态应用（Web、API）

❌ 不适合：

- 数据库（用 StatefulSet）

------

# 七、Deployment vs StatefulSet（简单对比）

| 特性     | Deployment | StatefulSet |
| -------- | ---------- | ----------- |
| Pod 名称 | 随机       | 固定        |
| 存储     | 不保证     | 稳定        |
| 启动顺序 | 无序       | 有序        |
| 场景     | Web服务    | 数据库      |

------

# 八、你下一步该怎么练（重点）

我建议你按这个顺序练：

### ✔ Step 1

自己写 nginx Deployment YAML

### ✔ Step 2

改 image → 观察滚动更新

### ✔ Step 3

故意写错 image → 练回滚

### ✔ Step 4

改 replicas → 看扩缩容

------

# 九、进阶（你准备面试时必须会）

如果你学到这里，我可以继续带你深入：

- readinessProbe / livenessProbe（面试高频）
- Deployment 发布策略（蓝绿 / 灰度）
- HPA 自动扩缩容
- 和 Service / Ingress 联动

------

# 🔥 给你一个挑战（检验是否真的会）

你可以试着回答我：

👉 **如果我把 replicas=3 的 Deployment 更新镜像，整个过程中最多有多少个 Pod？最少有多少可用？为什么？**

如果你答得出来，Deployment 你已经超过 70% 的人了。

------

如果你愿意，我可以带你进入下一阶段 👉
 **“Deployment + Service + Ingress = 完整上线一套系统”**