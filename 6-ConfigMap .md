# 一、ConfigMap 是什么（先打牢认知）

一句话理解：

👉 **ConfigMap = 把配置从镜像里“解耦”出来，单独管理**

比如你现在有一个 nginx：

```
server {
    listen 80;
    location / {
        return 200 "hello world";
    }
}
```

你有两种做法：

### ❌ 错误做法（很多新手会这样）

把配置写死在镜像里 → 每改一次配置都要重新 build 镜像

### ✅ 正确做法（ConfigMap）

👉 配置单独存 Kubernetes 里，用的时候“挂进去”

------

# 二、ConfigMap 的 3 种用法（非常重要）

这是面试高频点👇

------

## 1️⃣ 作为环境变量（最简单）

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "dev"
  APP_DEBUG: "true"
```

Pod 使用：

```
envFrom:
  - configMapRef:
      name: app-config
```

👉 容器里就能拿到：

```
echo $APP_ENV
```

------

## 2️⃣ 作为单个变量注入（精细控制）

```
env:
  - name: ENV
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_ENV
```

------

## 3️⃣ 挂载为文件（🔥最重要，企业常用）

```
volumeMounts:
  - name: config-volume
    mountPath: /etc/config

volumes:
  - name: config-volume
    configMap:
      name: app-config
```

👉 容器里变成文件：

```
/etc/config/APP_ENV
/etc/config/APP_DEBUG
```

------

# 三、🚀 实战：用 ConfigMap 控制 Nginx 页面（强烈建议你动手）

这是一个**面试加分项目级案例**

------

## 🎯 目标

👉 用 ConfigMap 动态控制 nginx 页面内容
 👉 不重建镜像就能改页面

------

## 第一步：创建配置文件

在你的 master 节点：

```
mkdir configmap-demo
cd configmap-demo

cat > index.html <<EOF
<h1>Hello ConfigMap</h1>
EOF
```

------

## 第二步：创建 ConfigMap

```
kubectl create configmap nginx-config \
  --from-file=index.html
```

查看：

```
kubectl get cm
kubectl describe cm nginx-config
```

------

## 第三步：创建 Deployment

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-configmap-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-configmap
  template:
    metadata:
      labels:
        app: nginx-configmap
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html-volume
        configMap:
          name: nginx-config
```

```
apiVersion: apps/v1              # API 版本：Deployment 属于 apps 组，v1 是稳定版本
kind: Deployment                 # 资源类型：声明这是一个 Deployment 控制器
metadata:
  name: nginx-configmap-demo     # Deployment 的名字，唯一标识这个对象

spec:
  replicas: 1                    # 期望副本数：只运行 1 个 Pod

  selector:                      # 【选择器】：Deployment 靠这个找到它管理的 Pod
    matchLabels:
      app: nginx-configmap       # 必须匹配 Pod 模板中的 labels，否则关联失败

  template:                      # 【Pod 模板】：描述要创建的 Pod 长什么样
    metadata:
      labels:
        app: nginx-configmap     # Pod 标签：必须和上面的 selector 完全一致！
                                 # 这是 Deployment 和 Pod 之间的"绑定契约"

    spec:
      containers:
      - name: nginx              # 容器名称
        image: nginx:1.25        # 镜像版本：1.25（稳定版）

        volumeMounts:            # 【挂载点声明】：把哪个卷挂到容器里的哪个路径
        - name: html-volume       # 卷的名字（引用下面 volumes 中定义的）
          mountPath: /usr/share/nginx/html  
                                 # 挂载路径：Nginx 默认放静态页面的目录
                                 # 挂载后，该目录下原有文件会被 ConfigMap 内容覆盖/合并

      volumes:                     # 【卷定义】：声明 Pod 使用哪些卷
      - name: html-volume          # 卷名称（和上面的 volumeMounts.name 对应）
        configMap:                 # 卷类型：ConfigMap（将配置数据作为文件挂载）
          name: nginx-config       # ConfigMap 的名字：集群中必须已存在这个 ConfigMap
```

**核心逻辑总结：**

plain

```plain
ConfigMap "nginx-config" 中的数据
           ↓
    以文件形式挂载到 Pod
           ↓
    目录：/usr/share/nginx/html
           ↓
    Nginx 直接提供这些文件作为静态页面
```

------

**关键注意事项：**

| 注意点                     | 说明                                                         |
| :------------------------- | :----------------------------------------------------------- |
| **ConfigMap 必须预先存在** | 如果 `nginx-config` 不存在，Pod 会创建失败，处于 `ContainerCreating` 状态，报 `ConfigMap not found` |
| **挂载会覆盖原目录**       | `/usr/share/nginx/html` 原本有 Nginx 默认的 `index.html`，挂载后会被 ConfigMap 中的同名文件覆盖。如果 ConfigMap 里没有 `index.html`，访问会报 403 |
| **只读挂载**               | ConfigMap 卷默认是只读的，容器内无法修改这些文件             |
| **热更新**                 | ConfigMap 更新后，**已挂载的文件不会自动更新**（除非使用 subPath，但 subPath 也不自动更新）。需要滚动更新 Pod 才能生效 |
| **路径必须是目录**         | `mountPath` 必须是目录路径，不能是单个文件路径（除非配合 `subPath` 使用） |

------

**面试常问：**

> "ConfigMap 挂载后，修改 ConfigMap 数据，容器里的文件会变吗？"

**答：** 直接挂载（不指定 `subPath`）时，Kubelet 会定期同步，**大约几十秒到几分钟后会更新**。但如果用了 `subPath`，**不会自动更新**，因为 `subPath` 是挂载时做的一次性绑定。

> "这个 Deployment 的 selector 和 Pod label 不一致会怎样？"

**答：** Deployment 会不断创建新 Pod，但认为没有 Pod 满足条件，于是无限创建，造成 **"孤儿 Pod" 泛滥**。这是新手常见错误。



应用：

```
kubectl apply -f nginx.yaml
```

------

## 第四步：访问测试

```
kubectl expose deployment nginx-configmap-demo --port=80 --type=NodePort
```

访问浏览器：

```
http://节点IP:NodePort
```

👉 你会看到：

```
Hello ConfigMap
```

------

## 第五步：🔥动态修改配置（重点）

修改本地文件：

```
echo "<h1>ConfigMap Updated!</h1>" > index.html
```

更新 ConfigMap：

```
kubectl create configmap nginx-config \
  --from-file=index.html \
  -o yaml --dry-run=client | kubectl apply -f -
```

------

## 观察变化：

👉 等一会（或刷新页面）
 👉 页面会变：

```
ConfigMap Updated!
```

💥 **重点结论：**

✔ 不用重启 Pod
 ✔ 不用重新构建镜像
 ✔ 配置实时生效（有延迟）

------

# 四、🔥 进阶知识（面试必问）

------

## ❗ ConfigMap 更新不会触发 Pod 重启

👉 Deployment 不会自动滚动更新

解决方案：

```
kubectl rollout restart deployment nginx-configmap-demo
```

或者：

👉 用 annotation 强制更新（高级玩法）

------

## ❗ 文件更新不是实时的（有延迟）

K8s 是通过：

👉 定期同步（默认约1分钟）

------

## ❗ ConfigMap vs Secret

| 类型      | 用途        |
| --------- | ----------- |
| ConfigMap | 普通配置    |
| Secret    | 密码、token |

------

# 五、🔥 面试加分总结（你可以直接背）

如果面试官问：

👉 “ConfigMap 有什么用？”

你可以这样说：

> ConfigMap 用于将应用配置与镜像解耦，支持通过环境变量或文件方式注入容器。
>  在实际生产中通常用于管理 nginx 配置、应用参数等。
>  通过 ConfigMap 可以在不重新构建镜像的情况下修改配置，提高运维效率。

------

# 六、我给你的下一步建议（非常关键）

你现在已经：

✅ 会 Deployment
 ✅ 会 StatefulSet
 ✅ 正在学 ConfigMap

👉 下一步应该：

### 🔥 做一个完整项目（面试杀手）

我可以带你做：

👉 **ConfigMap + Secret + MySQL + SpringBoot + Nginx**

或者更偏云原生：

👉 **ConfigMap + Ingress / Gateway API 做灰度发布**

------

# 👉 要不要我带你做一个“企业级实战项目”？（直接拉开差距）

我可以一步步带你做：

✔ 配置热更新
 ✔ 多环境 dev/test/prod
 ✔ 配合你现在的3台机器环境

你只需要说：

👉 **“上项目”** 或 **“继续进阶”**

明白，你是指 “访问浏览器” 这一部分。因为你是在本地的 Kubernetes 集群（3台虚拟机）上操作，这里访问的方式需要具体说明。我们详细拆解一下：

------

## 访问 Nginx Pod 的方法

### 1️⃣ 通过 NodePort 访问（最简单，适合学习/测试）

假设你在上一步创建了 Service：

```
kubectl expose deployment nginx-configmap-demo --port=80 --type=NodePort
```

1. 查看 NodePort 对应的端口：

```
kubectl get svc nginx-configmap-demo
```

输出示例：

```
NAME                      TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx-configmap-demo      NodePort   10.96.0.50     <none>        80:31234/TCP   2m
```

解释：

- `80` → Pod 内部端口
- `31234` → 你在浏览器访问的端口（随机生成的 NodePort）

1. 浏览器访问：

```
http://<任意节点的IP>:31234
```

- `<任意节点的IP>` 可以是你三台虚拟机中任意一台的内网 IP（比如 `192.168.36.100`）
- 端口用上面显示的 NodePort（这里是 `31234`）

------

### 2️⃣ 通过 Port Forward（本机临时访问）

如果你不想暴露 NodePort，可以临时转发本地端口：

```
kubectl port-forward deployment/nginx-configmap-demo 8080:80
```

然后在本机浏览器访问：

```
http://localhost:8080
```

------

### 3️⃣ 通过 Ingress（生产常用）

当你搭建了 Ingress Controller 或 Gateway API，可以用域名访问，但这个是进阶步骤。先掌握前两种就够。

------

💡 **小技巧：**

- NodePort 可以直接访问，但每次部署端口可能会变
- Port Forward 方便临时测试，适合实验室环境

------

如果你愿意，我可以帮你直接画一个**可视化访问路径图**，显示 Pod → Service → Node → 浏览器的完整流程，让你对 ConfigMap + Deployment 的访问链路一目了然。

你想让我画吗？