# `创建一个名叫 my-pod 的 Pod`

## 1. `apiVersion: v1`

- 表示使用 **Kubernetes 核心 API 版本 v1**
- Pod 属于最基础的资源，固定用 v1

## 2. `kind: Pod`

- **资源类型：Pod**
- Pod 是 K8s 里**最小的运行单位**，里面可以跑一个或多个容器

## 3. `metadata:` 元数据

```yaml
metadata:
  name: my-pod
```

- 给这个 Pod 起名字：**my-pod**
- 名字用于区分、管理、查看日志

## 4. `spec:` 规格（你要让 Pod 干什么）

这是核心部分，定义 Pod 里跑什么容器、怎么运行。

```yaml
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
```

### 逐行解释：

- `containers:`

  定义 Pod 里的**容器列表**

  

- `- name: nginx`

  给容器起名字：**nginx**

  

- `image: nginx:alpine`

  使用的**镜像**：

  `nginx` = 网页服务器

  `alpine` = 超轻量版

  

- `ports:`

  声明容器暴露的端口

  

- `containerPort: 80`

  容器内部开放 **80 端口**（nginx 默认网页端口）