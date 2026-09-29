# Helm 完整实战教程

## 一、Helm 概念

**Helm** 是 Kubernetes 的包管理工具，类比关系：

| 传统环境              | K8s 环境        |
| :-------------------- | :-------------- |
| apt/yum/brew          | Helm            |
| .deb/.rpm             | Chart           |
| /etc/apt/sources.list | Helm Repository |

**核心概念：**

- **Chart**：Helm 包，包含一组相关的 K8s 资源模板（Deployment、Service、ConfigMap 等）
- **Release**：Chart 的实例，同一个 Chart 可以安装多次，每次安装产生一个独立的 Release
- **Repository**：Chart 的远程仓库，类似 Maven Central 或 npm registry
- **Values**：自定义配置，通过 `values.yaml` 或 `--set` 注入模板，实现"一套模板，多套环境"

------

## 二、Helm 架构

plain

```plain
┌─────────────────┐
│   Helm Client   │  ← 用户交互层（helm install/upgrade/rollback）
│  (helm binary)  │
└────────┬────────┘
         │ HTTPS/gRPC
         ▼
┌─────────────────┐
│    Tiller v2    │  ← Helm 2 有服务端组件（已废弃）
│   (removed)     │
└─────────────────┘

Helm 3 架构（纯客户端）：
┌─────────────────────────────────────┐
│           Helm Client               │
│  ┌─────────┐ ┌─────────┐ ┌──────┐ │
│  │  CLI    │ │  Chart  │ │ Kube │ │
│  │ Layer   │ │ Loader  │ │Config│ │
│  └────┬────┘ └────┬────┘ └──┬───┘ │
│       └────────────┴─────────┘     │
│              │                     │
│              ▼                     │
│       ┌─────────────┐             │
│       │  Template   │ ← Go template + Sprig 函数库 │
│       │  Engine     │             │
│       └──────┬──────┘             │
│              │ render              │
│              ▼                    │
│       ┌─────────────┐             │
│       │  K8s API    │ ← 直接操作集群（~/.kube/config）│
│       │  Client     │             │
│       └─────────────┘             │
└─────────────────────────────────────┘
```

**Helm 3 关键改进：**

- 移除 Tiller，直接通过 kubeconfig 连接 K8s API Server
- 安全性提升（RBAC 直接生效，无需给 Tiller 开权限）
- Release 信息存储在 Secret 中（`sh.helm.release.v1.<release-name>.v<<version>`）
- 支持 JSON Schema 校验 values

------

## 三、常用命令与场景速查表

### 1. 仓库管理

```bash
# 添加仓库（场景：使用 bitnami 官方源）
helm repo add bitnami https://charts.bitnami.com/bitnami

# 添加国内镜像源（场景：网络慢或无法访问外网）
helm repo add bitnami-cn https://charts.bitnami.com/bitnami --force-update

# 列出所有仓库
helm repo list

# 更新本地索引（场景：安装前确保拿到最新 Chart 版本）
helm repo update

# 搜索 Chart（场景：找 redis 包）
helm search repo redis
helm search repo redis --versions    # 查看所有历史版本
```

### 2. 安装与卸载

```bash
# 基础安装（场景：快速部署测试环境）
helm install my-redis bitnami/redis

# 指定命名空间（场景：多租户隔离）
helm install my-redis bitnami/redis -n dev --create-namespace

# 自定义 values（场景：生产环境修改配置）
helm install my-redis bitnami/redis -f my-values.yaml

# 命令行覆盖单个值（场景：临时调试）
helm install my-redis bitnami/redis --set auth.enabled=false

# 卸载并保留历史（场景：清资源但保留回滚可能）
helm uninstall my-redis --keep-history

# 彻底卸载（场景：完全清理）
helm uninstall my-redis
```

### 3. 升级与回滚

```bash
# 升级 Release（场景：更新镜像版本或修改配置）
helm upgrade my-redis bitnami/redis -f my-values.yaml

# 强制重启（场景：ConfigMap 变了但 Deployment 没感知）
helm upgrade my-redis bitnami/redis --force

# 查看历史版本（场景：准备回滚前查看）
helm history my-redis

# 回滚到指定版本（场景：新版本出故障，紧急恢复）
helm rollback my-redis 2

# 升级时自动安装（场景：CI/CD 流水线，首次和更新同一命令）
helm upgrade --install my-redis bitnami/redis -f values.yaml
```

### 4. 调试与诊断

```bash
# 本地渲染模板（场景：不写集群，先看生成的 YAML 对不对）
helm template my-redis bitnami/redis -f my-values.yaml

# 安装前预演（场景：验证资源是否可创建，不实际部署）
helm install my-redis bitnami/redis --dry-run

# 获取已安装 Release 的 values（场景：排查线上配置）
helm get values my-redis
helm get values my-redis --all   # 包含默认值

# 查看 Release 状态
helm status my-redis

# 列出所有 Release（场景：查看集群里装了什么）
helm list --all-namespaces
```

------

## 四、Redis 一主一从 Helm 实战

### 前置准备

```bash
# 确认 Helm 版本
helm version

# 确认 kubectl 能连集群
kubectl cluster-info
```

### Step 1：添加并切换 Helm 源

bash

```bash
# 添加 Bitnami 官方源（最权威的 Chart 源）
helm repo add bitnami https://charts.bitnami.com/bitnami

# 如果网络不佳，可换国内源（如 Azure 中国镜像，需确认当前可用性）
# helm repo add bitnami https://mirror.azure.cn/kubernetes/charts/bitnami

# 更新索引
helm repo update

# 搜索 Redis Chart
helm search repo bitnami/redis --versions
```

### Step 2：拉取并修改配置（一主一从）

bash

```bash
# 拉取 Chart 到本地，方便查看和修改
helm pull bitnami/redis --untar
cd redis

# 查看默认 values
cat values.yaml | less
```

创建自定义配置文件 `redis-master-slave.yaml`：

yaml

```yaml
# ==========================================
# Redis 一主一从配置
# ==========================================

# 全局配置
global:
  # 关闭集群模式（我们要一主一从，不是 Cluster）
  redis:
    cluster:
      enabled: false

  # 密码配置（生产环境务必设置）
  auth:
    enabled: true
    password: "MySecurePassword123"

# 主节点配置
master:
  # 1 个主节点
  replicaCount: 1
  
  # 持久化配置
  persistence:
    enabled: true
    storageClass: "standard"    # 根据你的集群 StorageClass 修改
    size: 8Gi
    accessModes:
      - ReadWriteOnce

  # 资源限制（根据实际调整）
  resources:
    limits:
      memory: "512Mi"
      cpu: "500m"
    requests:
      memory: "256Mi"
      cpu: "250m"

  # 服务类型
  service:
    type: ClusterIP
    port: 6379

# 从节点配置
replica:
  # 1 个从节点（一主一从）
  replicaCount: 1
  
  # 从节点也开启持久化
  persistence:
    enabled: true
    storageClass: "standard"
    size: 8Gi

  # 资源限制
  resources:
    limits:
      memory: "512Mi"
      cpu: "500m"
    requests:
      memory: "256Mi"
      cpu: "250m"

# 网络策略（可选，测试环境可关闭）
networkPolicy:
  enabled: false

# 指标监控（可选）
metrics:
  enabled: true
  service:
    enabled: true
```

### Step 3：安装 Redis

bash

```bash
# 安装到 redis 命名空间
helm install redis-ha bitnami/redis \
  -f redis-master-slave.yaml \
  -n redis \
  --create-namespace

# 查看安装状态
helm status redis-ha -n redis

# 查看 Pod
kubectl get pods -n redis -w

# 预期输出：
# NAME                READY   STATUS    RESTARTS   AGE
# redis-ha-master-0   1/1     Running   0          2m
# redis-ha-replica-0  1/1     Running   0          2m
```

### Step 4：验证主从同步

bash

```bash
# 进入主节点
kubectl exec -it redis-ha-master-0 -n redis -- redis-cli -a MySecurePassword123

# 在主节点写入数据
127.0.0.1:6379> SET hello helm
OK
127.0.0.1:6379> exit

# 进入从节点验证同步
kubectl exec -it redis-ha-replica-0 -n redis -- redis-cli -a MySecurePassword123

# 从节点应该能读到数据（从节点默认只读）
127.0.0.1:6379> GET hello
"helm"

# 验证从节点角色
127.0.0.1:6379> INFO replication
# 应显示 role:slave，并指向 master
```

### Step 5：修改配置并升级

假设我们要**扩大 PVC 到 16Gi 并增加内存限制**：

bash

```bash
# 编辑配置文件
cat > redis-master-slave.yaml << 'EOF'
global:
  redis:
    cluster:
      enabled: false
  auth:
    enabled: true
    password: "MySecurePassword123"

master:
  replicaCount: 1
  persistence:
    enabled: true
    storageClass: "standard"
    size: 16Gi          # ← 从 8Gi 改为 16Gi
  resources:
    limits:
      memory: "1Gi"     # ← 从 512Mi 改为 1Gi
      cpu: "1000m"
    requests:
      memory: "512Mi"
      cpu: "500m"

replica:
  replicaCount: 1
  persistence:
    enabled: true
    storageClass: "standard"
    size: 16Gi
  resources:
    limits:
      memory: "1Gi"
      cpu: "1000m"
    requests:
      memory: "512Mi"
      cpu: "500m"

metrics:
  enabled: true
EOF

# 执行升级（关键命令！）
helm upgrade redis-ha bitnami/redis \
  -f redis-master-slave.yaml \
  -n redis

# 观察滚动更新
kubectl get pods -n redis -w
```

> ⚠️ **注意**：PVC 扩容依赖 StorageClass 是否支持 `allowVolumeExpansion`，如果不支持，可能需要手动处理。

### Step 6：回滚操作

bash

```bash
# 查看发布历史
helm history redis-ha -n redis

# 输出示例：
# REVISION  UPDATED                   STATUS      CHART        APP VERSION DESCRIPTION
# 1         Mon Jun 1 08:50:00 2026   superseded  redis 17.3.7 7.0.10      Install complete
# 2         Mon Jun 1 09:15:00 2026   deployed    redis 17.3.7 7.0.10      Upgrade complete

# 假设升级后发现问题（如内存配置过大导致调度失败），回滚到版本 1
helm rollback redis-ha 1 -n redis

# 验证回滚成功
helm status redis-ha -n redis
kubectl get pods -n redis
```

### Step 7：Helm 卸载 Redis

bash

```bash
# 方式 A：标准卸载（删除所有资源，保留 Release 历史）
helm uninstall redis-ha -n redis --keep-history

# 方式 B：彻底卸载（删除所有资源 + Release 历史）
helm uninstall redis-ha -n redis

# 验证清理
helm list -n redis              # 应该看不到 redis-ha
kubectl get all -n redis        # 应该没有残留

# 如果 PVC 需要保留数据，卸载前单独处理：
kubectl get pvc -n redis
# 手动删除或保留 PVC
```

------

## 五、实战技巧总结

| 场景             | 推荐命令                                                     |
| :--------------- | :----------------------------------------------------------- |
| 首次部署生产环境 | `helm install <name> <chart> -f values.yaml -n <ns> --create-namespace` |
| CI/CD 统一命令   | `helm upgrade --install <name> <chart> -f values.yaml`       |
| 调试模板不部署   | `helm template <name> <chart> -f values.yaml | less`         |
| 紧急故障恢复     | `helm rollback <name> <revision>`                            |
| 查看线上实际配置 | `helm get values <name> --all`                               |
| 清理环境彻底卸载 | `helm uninstall <name> && kubectl delete ns <ns>`            |

**Helm 3 最佳实践：**

1. **版本化 values**：`values-dev.yaml`、`values-prod.yaml`
2. **Chart 依赖**：在 `Chart.yaml` 中声明 `dependencies`，自动下载子 Chart
3. **Hooks**：利用 `helm.sh/hook` 实现安装前 Job（如数据库迁移）
4. **Secret 存储**：Release 信息存于 Secret，勿手动删除这些 Secret
5. **原子升级**：`helm upgrade --atomic` — 升级失败自动回滚

如有需要，我可以进一步展开 **Helm Chart 自定义开发** 或 **Helm + ArgoCD/GitOps 集成** 的内容。