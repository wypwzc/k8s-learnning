# Kubernetes HPA 详解

## 一、核心概念

**HPA（Horizontal Pod Autoscaler，水平 Pod 自动扩缩容）** 是 Kubernetes 内置的控制器，它根据观察到的指标（如 CPU、内存、自定义指标）自动调整 Pod 的副本数量。

### 与 VPA 的区别

| 特性     | HPA（水平扩缩）    | VPA（垂直扩缩）                |
| :------- | :----------------- | :----------------------------- |
| 调整方式 | 增加/减少 Pod 数量 | 调整单个 Pod 的 CPU/内存资源   |
| 适用场景 | 无状态服务、微服务 | 有状态服务、不可水平扩展的应用 |
| 触发条件 | 负载指标阈值       | 资源实际使用量                 |

### 工作原理

1. **Metrics Server** 采集 Pod 资源使用数据
2. **HPA Controller** 定期（默认 15 秒）计算目标副本数
3. **Deployment/ReplicaSet** 根据计算结果调整副本数

计算公式：

```plain
desiredReplicas = ceil[currentReplicas * (currentMetricValue / desiredMetricValue)]
```

------

## 二、使用场景

### 1. 流量波动明显的 Web 服务

- **场景**：电商大促、新闻热点、社交应用
- **价值**：高峰期自动扩容，低峰期自动缩容，避免资源浪费

### 2. 定时任务/批处理

- **场景**：夜间数据处理、报表生成
- **价值**：根据队列长度自动调整消费 Pod 数量

### 3. 微服务架构

- **场景**：某个服务成为瓶颈（如支付服务、推荐服务）
- **价值**：仅对热点服务扩容，不影响整体集群

### 4. 成本优化

- **场景**：云原生环境按量付费
- **价值**：避免为应对峰值而长期预留大量资源

------

## 三、完整示例

### 前置条件：安装 Metrics Server

```bash
# 1. 下载官方 YAML
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 2. 替换为阿里云镜像（或其他国内源）
sed -i 's|registry.k8s.io/metrics-server|registry.aliyuncs.com/google_containers|g' components.yaml

# 3. 部署
kubectl apply -f components.yaml
```

**Metrics Server 默认会验证 kubelet 的 TLS 证书**，在测试环境或证书配置不完整的情况下，Pod 会报错无法启动。需要在容器启动参数里加上 `--kubelet-insecure-tls` 跳过验证。

##### 修改方法

##### 修改已下载的 YAML 文件

找到 `containers.args` 部分，添加参数：

yaml

```yaml
# components.yaml 片段
spec:
  containers:
  - name: metrics-server
    image: registry.aliyuncs.com/google_containers/metrics-server:v0.7.1
    args:
      - --cert-dir=/tmp
      - --secure-port=10250
      - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
      - --kubelet-use-node-status-port
      - --metric-resolution=15s
      - --kubelet-insecure-tls    # ← 加上这一行，跳过 kubelet TLS 验证
```

然后重新部署：

```bash
kubectl apply -f components.yaml
```

### 步骤 1：部署示例应用（Deployment + Service）

```yaml
# app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-app
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "100m"      # 必须设置，HPA 基于 request 计算
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

部署：

```bash
kubectl apply -f app-deployment.yaml
```

### 步骤 2：创建 HPA 资源

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-app
  minReplicas: 2          # 最少保留 2 个 Pod
  maxReplicas: 10         # 最多扩展到 10 个 Pod
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50   # CPU 平均利用率超过 50% 时扩容
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70   # 内存平均利用率超过 70% 时扩容
  behavior:               # 可选：控制扩缩容速度
    scaleDown:
      stabilizationWindowSeconds: 300  # 缩容前等待 5 分钟，避免抖动
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60   # 每分钟最多缩容 50%
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15   # 每 15 秒最多扩容 100%
```

部署：

bash

```bash
kubectl apply -f hpa.yaml
```

### 步骤 3：验证与压测

查看 HPA 状态：

bash

```bash
kubectl get hpa nginx-hpa
# 输出示例：
# NAME        REFERENCE              TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
# nginx-hpa   Deployment/nginx-app   0%/50%     2         10        2          10m
```

模拟压力测试（扩容验证）：

bash

```bash
# 进入集群内压测
kubectl run -it --rm load-generator --image=busybox:1.28 --restart=Never -- /bin/sh -c "
while sleep 0.01; do
  wget -q -O- http://nginx-service
done
"
```

观察扩容过程：

```bash
kubectl get hpa nginx-hpa -w
# 你会看到 TARGETS 上升，REPLICAS 从 2 逐渐增加到 10
```

停止压测后观察缩容：

```bash
# 等待约 5 分钟（stabilizationWindowSeconds），Pod 数量会逐渐回落到 2
kubectl get deployment nginx-app -w
```

------

## 四、进阶：基于自定义指标（KEDA / Prometheus）

如果基于 CPU/内存不够，还可以基于**业务指标**扩容，比如：

```yaml
# 基于 Prometheus 自定义指标（需安装 prometheus-adapter）
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"   # 每秒请求超过 1000 时扩容
```

或使用 **KEDA**（更推荐的方案）：

yaml

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: queue-scaler
spec:
  scaleTargetRef:
    name: worker-app
  pollingInterval: 10
  minReplicaCount: 0        # 甚至可以缩到 0！
  maxReplicaCount: 20
  triggers:
  - type: redis
    metadata:
      address: redis:6379
      listName: my-queue
      listLength: "100"     # 队列长度超过 100 时扩容
```

------

## 五、最佳实践

| 建议                            | 说明                                    |
| :------------------------------ | :-------------------------------------- |
| **必须设置 resources.requests** | HPA 依赖 request 计算利用率             |
| **不要同时用 HPA + VPA**        | 两者冲突，建议先用 HPA                  |
| **设置合适的缩容稳定窗口**      | 避免流量抖动导致反复扩缩                |
| **配合 Cluster Autoscaler**     | HPA 扩容后若节点不足，CA 自动加节点     |
| **监控 HPA 事件**               | `kubectl describe hpa xxx` 查看扩容历史 |

