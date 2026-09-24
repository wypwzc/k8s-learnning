### statefulset

灰度发布   滚动更新partition

kubectl scale sts(statefulset) web --replicas=5  #使副本数变为5

kubectl edit sts web   #修改partition为3 改镜像版本 

更新完以后只有3 4更新了镜像 0 1 2并没有发生改变

## 一、什么是 StatefulSet（本质）

在 Kubernetes 里：

- **Deployment**：适合无状态（Web服务、API）
- **StatefulSet**：适合有状态（数据库、消息队列、缓存）

👉 StatefulSet 解决的核心问题：

1. **Pod 有固定身份（不会乱变）**
2. **有稳定的网络名称**
3. **每个 Pod 绑定独立存储**
4. **有序启动 / 有序删除**

------

## 二、为什么要用 StatefulSet

你想象一个数据库集群（比如 MySQL、Redis）：

如果用 Deployment：

- Pod 名字随机（mysql-abc123 ❌）
- 重建后 IP 变了 ❌
- 存储可能丢 ❌

👉 这对数据库是灾难

------

用 StatefulSet：

- Pod 名字固定：

  ```
  mysql-0
  mysql-1
  mysql-2
  ```

- 永远按顺序编号

- 每个实例都有自己磁盘

------

## 三、核心特性（面试必问）

### 1️⃣ 稳定 Pod 名字

格式：

```
<statefulset-name>-<ordinal>
```

比如：

```
nginx-0
nginx-1
nginx-2
```

👉 删除再创建，还是原来的名字

------

### 2️⃣ 稳定网络身份（DNS）

结合 Headless Service：

```
nginx-0.nginx.default.svc.cluster.local
```

👉 每个 Pod 都有独立域名（非常关键）

------

### 3️⃣ 独立存储（PVC）

每个 Pod 都会自动创建：

```
data-nginx-0
data-nginx-1
data-nginx-2
```

👉 数据不会串

------

### 4️⃣ 有序部署 & 删除

启动顺序：

```
nginx-0 → nginx-1 → nginx-2
```

删除顺序：

```
nginx-2 → nginx-1 → nginx-0
```

👉 保证集群初始化安全

------

## 四、核心组件结构

一个 StatefulSet 一般包含：

1. StatefulSet
2. Headless Service（必须）
3. PVC（自动或手动）

------

## 五、最小示例（你一定要会写）

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  clusterIP: None   # 👈 Headless Service
  selector:
    app: nginx
  ports:
    - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx
spec:
  serviceName: nginx
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
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

------

## 六、你必须理解的关键点（很多人卡在这里）

### ❗为什么必须 Headless Service？

普通 Service：

- 负载均衡（随机访问）

Headless：

- 返回 Pod 的真实 IP
- 用于 **点对点通信**

👉 StatefulSet 就靠它做 DNS

------

### ❗StatefulSet ≠ Deployment + PVC

很多人误解：

👉 “我给 Deployment 加 PVC 不就行了？”

❌ 错，因为：

- 没有固定 Pod 名
- 没有有序启动
- 没有稳定网络身份

------

## 七、适用场景（重点）

你可以记住一句话：

👉 **只要涉及“数据 + 集群”，优先考虑 StatefulSet**

常见：

- MySQL / PostgreSQL
- Redis Cluster
- Kafka
- Zookeeper
- Elasticsearch

------

## 八、面试高频对比

| 特性     | Deployment | StatefulSet |
| -------- | ---------- | ----------- |
| Pod 名字 | 随机       | 固定        |
| 网络     | 不稳定     | 稳定 DNS    |
| 存储     | 共享/临时  | 独立 PVC    |
| 启动顺序 | 无序       | 有序        |
| 使用场景 | Web/API    | 数据库      |

------

## 九、进阶你下一步该学什么

如果你是计算机专业，我建议你继续往下：

1. Headless Service 深入
2. PVC / PV / StorageClass
3. StatefulSet + MySQL 实战
4. Operator（比如 MySQL Operator）

------

如果你愿意，我可以带你做一个**真实项目**：

👉 用 StatefulSet 部署一个 **MySQL 主从集群**（非常加分，面试直接拉开差距）

要不要我一步一步带你搭？
