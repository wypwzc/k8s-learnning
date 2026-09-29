# StatefulSet 部署 MySQL 主从集群

# （企业级思路）

这是你从：

> “会 Kubernetes”

到：

> “懂云原生数据库部署”

的重要一步。

很多公司面试：

- MySQL 主从
- Redis 集群
- Kafka
- Elasticsearch

本质全是：

👉 StatefulSet + Headless Service + PVC

你把这个吃透，后面全通。

------

# 一、最终效果

我们会部署：

```
mysql-0  （主库）
mysql-1  （从库）
mysql-2  （从库）
```

特点：

- Pod 名字固定
- 每个 Pod 独立存储
- 自动 DNS
- 主从同步

------

# 二、先理解架构（非常重要）

## StatefulSet 网络结构

```
pod-name.service-name
```

StatefulSet：

```
mysql-0.mysql
mysql-1.mysql
mysql-2.mysql
```

其中：

```
mysql
```

来自：

```
serviceName: mysql
```

------

## Headless Service

必须：

```
clusterIP: None
```

意思：

```
不要给我分配 VIP
不要负载均衡
```

这样 DNS 才会返回 Pod IP。

------

# 三、准备环境

你已经有：

- 1 master
- 2 node

完全够。

先确认：

```
kubectl get nodes
```

全部 Ready。

------

# 四、创建 Namespace

```
kubectl create ns mysql
```

查看：

```
kubectl get ns
```

------

# 五、创建 Headless Service（必须）

创建：

```
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: mysql
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
    name: mysql
```

保存：

```
vim mysql-headless.yaml
```

应用：

```
kubectl apply -f mysql-headless.yaml
```

查看：

```
kubectl get svc -n mysql
```

你会看到：

```
CLUSTER-IP   None
```

这说明是 Headless Service。

------

# 六、创建 StatefulSet

这是核心。

创建：

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: mysql
spec:
  serviceName: mysql
  replicas: 3

  selector:
    matchLabels:
      app: mysql

  template:
    metadata:
      labels:
        app: mysql

    spec:
      containers:
      - name: mysql
        image: mysql:5.7

        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "123456"

        ports:
        - containerPort: 3306
          name: mysql

        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql

  volumeClaimTemplates:
  - metadata:
      name: mysql-data

    spec:
      accessModes: [ "ReadWriteOnce" ]

      resources:
        requests:
          storage: 1Gi
```

```
apiVersion: apps/v1
kind: StatefulSet

metadata:
  name: mysql                # StatefulSet 名字
  namespace: mysql           # 所属命名空间

spec:

  serviceName: mysql         # 关联的 Headless Service 名字
                               # 必须对应：
                               # kind: Service
                               # metadata:
                               #   name: mysql
                               #
                               # 用于生成稳定 DNS：
                               # mysql-0.mysql
                               # mysql-1.mysql

  replicas: 3                # 创建 3 个 Pod
                               # 最终：
                               # mysql-0
                               # mysql-1
                               # mysql-2

  selector:
    matchLabels:
      app: mysql             # StatefulSet 管理带有 app=mysql 标签的 Pod

  template:                  # Pod 模板（定义 Pod 长什么样）

    metadata:
      labels:
        app: mysql           # Pod 标签
                               # 必须和 selector.matchLabels 一致

    spec:

      containers:

      - name: mysql          # 容器名字

        image: mysql:5.7     # 使用 mysql 5.7 镜像

        env:

        - name: MYSQL_ROOT_PASSWORD
          value: "123456"    # 初始化 MySQL root 密码
                               # MySQL 官方镜像启动时会读取该环境变量

        ports:

        - containerPort: 3306
          name: mysql        # 容器监听 3306 端口
                               # 注意：
                               # 这里只是声明容器端口
                               # 不是端口映射

        volumeMounts:

        - name: mysql-data   # 挂载存储卷
          mountPath: /var/lib/mysql
                               # MySQL 数据目录
                               # 数据会保存到这里

  volumeClaimTemplates:      # PVC 模板（StatefulSet 核心）

  - metadata:
      name: mysql-data       # PVC 名字
                               # 必须和 volumeMounts.name 对应

    spec:

      accessModes:
      - ReadWriteOnce        # 只能被一个节点挂载读写
                               # 数据库最常见模式

      resources:

        requests:
          storage: 1Gi       # 每个 Pod 申请 1Gi 存储

                               # StatefulSet 会自动创建：

                               # mysql-data-mysql-0
                               # mysql-data-mysql-1
                               # mysql-data-mysql-2

                               # 每个 Pod 独立存储
                               # 删除 Pod 数据不会丢
```

保存：

```
vim mysql-sts.yaml
```

部署：

```
kubectl apply -f mysql-sts.yaml
```

------

# 七、观察 StatefulSet 特性（重点）

查看：

```
kubectl get pods -n mysql -w
```

你会看到：

```
mysql-0
mysql-1
mysql-2
```

并且：

```
mysql-0 Running 后
mysql-1 才会创建
```

这就是：

👉 有序启动

------

# 八、查看 PVC（非常重要）

执行：

```
kubectl get pvc -n mysql
```

你会看到：

```
mysql-data-mysql-0
mysql-data-mysql-1
mysql-data-mysql-2
```

这说明：

👉 每个 Pod 有独立存储

这就是 StatefulSet 最核心的地方。

------

# 九、验证 DNS（超级重点）

进入 Pod：

```
kubectl exec -it mysql-0 -n mysql -- bash
```

测试：

```
ping mysql-1.mysql
```

成功说明：

```
mysql-1.mysql
```

这个 DNS 生效了。

------

# 十、理解 DNS 原理（面试高频）

格式：

```
pod-name.service-name.namespace.svc.cluster.local
```

例如：

```
mysql-0.mysql.mysql.svc.cluster.local
```

拆开：

| 部分              | 含义      |
| ----------------- | --------- |
| mysql-0           | Pod       |
| mysql             | Service   |
| mysql             | Namespace |
| svc.cluster.local | 集群域名  |

------

# 十一、为什么数据库必须 StatefulSet

因为数据库需要：

| 功能     | StatefulSet |
| -------- | ----------- |
| 固定身份 | ✔           |
| 固定 DNS | ✔           |
| 独立磁盘 | ✔           |
| 有序启动 | ✔           |

Deployment 做不到。

------

# 十二、现在还缺什么？

目前只是：

```
3 个独立 MySQL
```

还不是：

```
主从复制
```

下一步：

我们需要：

## mysql-0 作为 master

开启：

```
log-bin
server-id=1
```

------

## mysql-1/mysql-2 作为 slave

配置：

```
CHANGE MASTER TO
```

实现复制。

------

# 十三、企业真实情况

企业不会手工配主从。

而是：

- Operator
- Helm
- MySQL Operator
- Percona Operator

例如：

- Percona Operator
- Oracle MySQL Operator

但：

👉 你必须先懂 StatefulSet 原理。

否则永远只是：

```
helm install 一把梭
```

不知道底层。

------

# 十四、下一步（真正进入高级）

下面才是真正高级内容：

## 1️⃣ ConfigMap 管理 MySQL 配置

例如：

```
server-id
log-bin
```

------

## 2️⃣ initContainer 自动初始化主从

这是企业重点。

------

## 3️⃣ xtrabackup 数据同步

真正生产级。

------

## 4️⃣ Operator 原理

这是 K8s 高级开发方向。

------

下一步我可以继续带你：

# 「K8s StatefulSet + MySQL 主从复制真正完整实战」

包括：

- 自动区分 master/slave
- 自动生成 server-id
- 自动同步
- initContainer
- ConfigMap
- 主从复制原理
- 故障恢复

这个已经属于：

👉 中高级 DevOps / 云原生数据库运维

要不要继续？







# 你必须记住一句话（面试高频）

## 为什么 StatefulSet 必须 Headless Service？

答案：

```
因为 StatefulSet 需要稳定网络标识，
而 Headless Service 会直接返回 Pod IP，
从而实现 Pod 级别 DNS。
```

这句话很多面试官直接点头。

------

# 你再深入一点（高级）

Headless Service 本质上：

```
不做 kube-proxy 转发
```

所以：

- 没有 VIP
- 没有负载均衡
- DNS 直接解析 endpoints

你可以查看：

```
kubectl get endpoints -n mysql
```

会看到：

```
mysql-0 IP
mysql-1 IP
mysql-2 IP
```

------

# 企业里哪些东西依赖 Headless Service

非常多：

| 组件          | 是否使用 |
| ------------- | -------- |
| MySQL 集群    | ✔        |
| Redis Cluster | ✔        |
| Kafka         | ✔        |
| Elasticsearch | ✔        |
| ZooKeeper     | ✔        |
| Etcd          | ✔        |

因为：

这些组件都需要：

```
节点互相发现
```