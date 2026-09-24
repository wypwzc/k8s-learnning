很多人学到这里会混乱：

```
NFS
PV
PVC
StorageClass
```

感觉都是存储，但不知道谁是谁。

实际上它们属于不同层次。

------

# 一、先理解为什么会有 NFS

假设你部署了一个 MySQL：

```
Pod
 └── /var/lib/mysql
```

数据都存在容器里。

问题来了：

```
Pod 删除
↓
重新创建
↓
数据没了
```

因为容器文件系统是临时的。

所以 Kubernetes 需要：

```
容器重建
↓
数据还在
```

这就需要：

```
持久化存储（Persistent Storage）
```

------

# 二、NFS 是什么

NFS：

```
Network File System
网络文件系统
```

可以理解成：

```
远程共享磁盘
```

例如：

```
192.168.10.100
```

这台机器提供：

```
/data/nfs
```

目录。

然后：

```
Node1
Node2
Node3
```

都能挂载。

示意图：

```
                NFS Server
            192.168.10.100

               /data/nfs
                    │
      ┌─────────────┼─────────────┐
      │             │             │
    Node1         Node2         Node3
```

所有节点看到的是同一份数据。

例如：

```
Node1写入

hello.txt
```

Node2：

```
立即看到 hello.txt
```

------

# 三、为什么 Kubernetes 喜欢 NFS

因为 Pod 会漂移。

今天：

```
PodA
↓
Node1
```

明天：

```
PodA
↓
Node2
```

如果使用本地磁盘：

```
Node1:/data
```

Pod 到 Node2 后：

```
数据没了
```

但 NFS：

```
Node1
Node2
Node3
```

挂载同一个目录：

```
/data/nfs/mysql
```

无论 Pod 在哪：

```
数据都在
```

------

# 四、PV 是什么

PV：

```
PersistentVolume
持久卷
```

PV 本质：

```
Kubernetes 对存储资源的抽象
```

NFS 是真实存储。

PV 是 Kubernetes 里的描述对象。

关系：

```
NFS
 ↓
PV
```

例如：

```
apiVersion: v1
kind: PersistentVolume

metadata:
  name: nfs-pv

spec:
  capacity:
    storage: 5Gi

  accessModes:
    - ReadWriteMany

  nfs:
    path: /data/nfs
    server: 192.168.10.100
```

意思：

```
K8s：
我这里有一个存储资源

来自：
192.168.10.100:/data/nfs

大小：
5G
```

------

# 五、PVC 是什么

PVC：

```
PersistentVolumeClaim
持久卷申请
```

可以理解为：

```
申请单
```

开发人员不会直接用 PV。

而是：

```
我要10G存储
```

提交 PVC。

例如：

```
apiVersion: v1
kind: PersistentVolumeClaim

metadata:
  name: mysql-pvc

spec:
  accessModes:
    - ReadWriteMany

  resources:
    requests:
      storage: 1Gi
```

意思：

```
K8s
我想申请一个：

1G
RWX
存储
```

------

# 六、PV 和 PVC 的关系

非常像：

```
房东 ←→ 租客
```

------

PV：

```
房子
```

PVC：

```
租房申请
```

------

流程：

```
管理员创建PV
        ↓
开发创建PVC
        ↓
K8s自动匹配
        ↓
Bound
```

示意图：

```
PV(nfs-pv)
      │
      │ Bound
      ▼
PVC(mysql-pvc)
```

查看：

```
kubectl get pv
kubectl get pvc
```

看到：

```
STATUS=Bound
```

说明成功绑定。

------

# 七、Pod 怎么使用 PVC

Pod 不直接使用 NFS。

也不直接使用 PV。

而是：

```
Pod
 ↓
PVC
 ↓
PV
 ↓
NFS
```

这是 Kubernetes 标准链路。

------

示意图：

```
Pod
 │
 ▼
PVC
 │
 ▼
PV
 │
 ▼
NFS Server
```

------

例如：

```
apiVersion: v1
kind: Pod

metadata:
  name: nginx

spec:
  containers:
  - name: nginx
    image: nginx

    volumeMounts:
    - name: web-data
      mountPath: /usr/share/nginx/html

  volumes:
  - name: web-data
    persistentVolumeClaim:
      claimName: mysql-pvc
```

------

最终效果：

```
容器目录

/usr/share/nginx/html

↓ 实际映射

PVC

↓
PV

↓
NFS

↓
192.168.10.100:/data/nfs
```

------

# 八、为什么面试总是 NFS + PV + PVC 一起问

因为它们刚好构成完整链路：

```
真实存储
   ↓
NFS
   ↓
PV
   ↓
PVC
   ↓
Pod
```

面试官最喜欢让你画这个图：

```
                 NFS Server
                       │
                       ▼
                 PersistentVolume
                       │
                       ▼
            PersistentVolumeClaim
                       │
                       ▼
                     Pod
```

如果你能把这张图讲清楚：

```
NFS 是存储

PV 是存储资源

PVC 是存储申请

Pod 使用 PVC

PVC 绑定 PV

PV 指向 NFS
```

基本已经达到 Kubernetes 存储入门要求。

------

# 九、生产环境为什么后来又出现 StorageClass

早期模式：

```
管理员创建PV
开发创建PVC
```

每来一个 PVC：

```
都要手工创建PV
```

很麻烦。

于是出现：

```
StorageClass
```

变成：

```
PVC
 ↓
StorageClass
 ↓
自动创建PV
 ↓
NFS
```

即：

```
动态供给（Dynamic Provisioning）
```

现代 Kubernetes 基本都是：

```
NFS + StorageClass + PVC
```

而不是手工创建 PV。

------

你当前学习路线如果到了存储阶段，建议顺序是：

```
1. Volume
      ↓
2. emptyDir
      ↓
3. hostPath
      ↓
4. PV
      ↓
5. PVC
      ↓
6. NFS
      ↓
7. StorageClass
      ↓
8. NFS Dynamic Provisioner
      ↓
9. StatefulSet + PVC
```

学完第 9 步后，你就能理解为什么 MySQL、Redis、Kafka 等有状态应用都要和 PVC 绑定，以及 StatefulSet 为什么会自动创建 PVC。