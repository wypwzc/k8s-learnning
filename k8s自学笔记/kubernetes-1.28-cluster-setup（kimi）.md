# Kubernetes 1.28 集群搭建手册（1 Master + 2 Worker）

> **适用场景**：测试/生产环境快速部署  
> **适用系统**：CentOS 7/8/9、RHEL 8/9、Ubuntu 20.04/22.04  
> **目标版本**：Kubernetes v1.28.x + containerd + Flannel/Calico  
> **网络环境**：同时提供**国内镜像源**与官方源配置，适配无公网/有代理环境

---

## 一、环境要求与规划

| 角色 | 主机名 | IP 地址 | 最低配置 |
|------|--------|---------|----------|
| Master | `k8s-master` | `192.168.1.10` | 2C 2G 20G |
| Worker-1 | `k8s-node1` | `192.168.1.11` | 2C 2G 20G |
| Worker-2 | `k8s-node2` | `192.168.1.12` | 2C 2G 20G |

**所有节点必须满足**：
- 互通（内网 IP 互通，建议关闭外网防火墙或在安全组放行端口）
- 能够访问外网拉取镜像（或被墙时配置国内镜像源）
- **系统时间必须同步**（NTP/Chrony），时间偏差 > 1min 会导致 TLS 证书校验失败

---

## 二、基础环境配置（所有节点执行）

### 2.1 配置主机名与 hosts

**根据实际 IP 修改，3 台全部执行：**

```bash
# 在 Master 上执行
hostnamectl set-hostname k8s-master

# 在 Worker-1 上执行
hostnamectl set-hostname k8s-node1

# 在 Worker-2 上执行
hostnamectl set-hostname k8s-node2
```

**写入 hosts（3 台全部执行）：**

```bash
cat <<'EOF' >> /etc/hosts
192.168.1.10 k8s-master
192.168.1.11 k8s-node1
192.168.1.12 k8s-node2
EOF
```

### 2.2 关闭 Swap、SELinux、防火墙（学习环境）

> **生产环境**：不要直接 `systemctl stop firewalld`，应放行具体端口（见文末第十章）。

```bash
# 关闭 Swap（K8s 强制要求）
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab

# 关闭 SELinux（CentOS/RHEL 必须，否则 kubelet 无法读取容器日志）
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

# 关闭防火墙（仅学习环境）
systemctl stop firewalld
systemctl disable firewalld
```

### 2.3 时间同步（极易被忽视，但必做）

```bash
# CentOS / RHEL
yum install -y chrony
systemctl enable --now chronyd

# Ubuntu
apt update
apt install -y chrony
systemctl enable --now chronyd

# 验证时间同步
chronyc tracking
```

### 2.4 加载内核模块与网络参数

```bash
cat <<'EOF' > /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat <<'EOF' > /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system
```

---

## 三、安装容器运行时 containerd（所有节点执行）

Kubernetes 1.24+ **不再支持 Docker 作为运行时**，必须使用 containerd、CRI-O 等 CRI 运行时。本文使用 containerd。

### 3.1 添加 containerd 软件源并安装

#### CentOS / RHEL

```bash
# 安装 yum 工具
yum install -y yum-utils

# 添加 Docker 官方源（containerd.io 在此源中）
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 安装 containerd
yum install -y containerd.io
```

#### Ubuntu

```bash
# 安装依赖
apt update
apt install -y ca-certificates curl gnupg

# 添加 GPG 密钥
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

# 添加源
echo   "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg]   https://download.docker.com/linux/ubuntu   "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable"   > /etc/apt/sources.list.d/docker.list

# 安装
apt update
apt install -y containerd.io
```

### 3.2 配置 containerd（关键步骤）

```bash
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

# 1. 开启 systemd cgroup（与 kubelet 保持一致，否则 Pod 会频繁重启）
sed -i 's|SystemdCgroup = false|SystemdCgroup = true|g' /etc/containerd/config.toml

# 2. 替换 pause 镜像为国内源（保留原版本号，避免版本不一致导致重复拉取）
sed -i 's|registry.k8s.io/pause:|registry.aliyuncs.com/google_containers/pause:|g' /etc/containerd/config.toml

# 3. 启动 containerd
systemctl enable containerd
systemctl restart containerd
```

### 3.3 预拉取 pause 镜像（避免后续被墙）

```bash
# 从配置文件中提取实际使用的 pause 镜像地址和版本
PAUSE_IMAGE=$(grep 'sandbox_image' /etc/containerd/config.toml | sed 's/.*= "//;s/"$//' | tr -d ' ')
ctr -n k8s.io image pull "$PAUSE_IMAGE"
```

### 3.4 安装 crictl 并配置 endpoint（排查问题必备）

```bash
VERSION=$(kubectl version --client 2>/dev/null | grep -oP 'GitVersion:"v\K[0-9.]+' || echo "1.28.0")
# 如果上面获取失败，直接指定版本
[ -z "$VERSION" ] && VERSION="1.28.0"

wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v${VERSION}/crictl-v${VERSION}-linux-amd64.tar.gz
tar zxvf crictl-v${VERSION}-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-v${VERSION}-linux-amd64.tar.gz

# 配置默认 endpoint
cat <<'EOF' > /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```

---

## 四、安装 Kubernetes 组件（所有节点执行）

### 4.1 添加 Kubernetes 软件源

#### CentOS / RHEL（阿里云镜像）

```bash
cat <<'EOF' > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
EOF

yum makecache
```

> 如果坚持开启 GPG 校验，将 `gpgcheck=0` 改为 `1`，并添加：  
> `gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg`

#### Ubuntu（阿里云镜像）

```bash
apt update
apt install -y apt-transport-https ca-certificates curl

curl -fsSL https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main'   > /etc/apt/sources.list.d/kubernetes.list

apt update
```

### 4.2 安装并锁定版本

```bash
# CentOS / RHEL
yum install -y kubelet-1.28.2 kubeadm-1.28.2 kubectl-1.28.2
yum install -y kubernetes-cni  # 确保 /opt/cni/bin 有插件
yum versionlock add kubelet kubeadm kubectl  # 防止自动升级

# Ubuntu
apt install -y kubelet=1.28.2-00 kubeadm=1.28.2-00 kubectl=1.28.2-00
apt-mark hold kubelet kubeadm kubectl
```

### 4.3 启动 kubelet

```bash
systemctl enable kubelet
# 此时不要手动 start，等 kubeadm init/join 后它会自动拉起
```

> **注意**：安装后 `systemctl status kubelet` 会看到不断 **crash-loop**，这是**正常现象**。kubelet 在加入集群前没有配置，会反复退出，等 `kubeadm init` 或 `kubeadm join` 后自动恢复。

---

## 五、Master 节点初始化（仅在 Master 执行）

### 5.1 预拉取控制平面镜像（国内环境强烈推荐）

```bash
kubeadm config images pull   --kubernetes-version=v1.28.2   --image-repository=registry.aliyuncs.com/google_containers
```

### 5.2 初始化集群

```bash
kubeadm init   --apiserver-advertise-address=192.168.1.10   --kubernetes-version=v1.28.2   --pod-network-cidr=10.244.0.0/16   --image-repository=registry.aliyuncs.com/google_containers   --upload-certs
```

参数说明：
- `--pod-network-cidr=10.244.0.0/16`：**必须与 CNI 插件网段匹配**。Flannel 默认用 `10.244.0.0/16`，Calico 默认用 `192.168.0.0/16`。如果改 Calico，这里也要同步改。
- `--upload-certs`：将证书上传到 etcd，方便后续高可用扩展（单 Master 可选）。

### 5.3 配置 kubectl 访问权限

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

### 5.4 保存 join 命令

初始化成功后会输出类似：

```bash
kubeadm join 192.168.1.10:6443 --token abcdef.0123456789abcdef     --discovery-token-ca-cert-hash sha256:xxxxxxxx...
```

**务必保存**，Worker 节点需要用它加入集群。

> **Token 24 小时后会过期**。如果丢失，在 Master 上重新生成：  
> `kubeadm token create --print-join-command`

---

## 六、部署 CNI 网络插件（仅在 Master 执行）

**必须安装**，否则节点状态永远 `NotReady`，CoreDNS 也会 Pending。

### 方案 A：Flannel（简单、经典）

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

国内加速备用：
```bash
kubectl apply -f https://mirrors.aliyun.com/k8s-yaml/flannel/kube-flannel.yml
```

> 确保 Master 初始化时的 `--pod-network-cidr` 为 `10.244.0.0/16`。

### 方案 B：Calico（功能丰富、策略控制强）

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

> 使用 Calico 时，Master 初始化应改为 `--pod-network-cidr=192.168.0.0/16`。

---

## 七、Worker 节点加入集群（2 台 Worker 执行）

### 7.1 执行 join 命令

使用 Master 初始化时保存的命令：

```bash
kubeadm join 192.168.1.10:6443 --token abcdef.0123456789abcdef     --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 7.2 如果 token 过期或丢失

在 **Master** 上执行：

```bash
kubeadm token create --print-join-command
```

复制输出的新命令到 Worker 执行即可。

---

## 八、集群验证（Master 执行）

### 8.1 查看节点状态

```bash
kubectl get nodes
```

预期结果（约 1-2 分钟后全部 Ready）：

```
NAME         STATUS   ROLES           AGE   VERSION
k8s-master   Ready    control-plane   5m    v1.28.2
k8s-node1    Ready    <none>          2m    v1.28.2
k8s-node2    Ready    <none>          2m    v1.28.2
```

### 8.2 查看系统 Pod

```bash
kubectl get pods -n kube-system
kubectl get pods -n kube-flannel  # 如果用了 Flannel
```

所有 Pod 状态应为 `Running`。

### 8.3 测试 DNS 与网络

```bash
kubectl run test --image=busybox:1.36 --restart=Never -- sleep 3600
kubectl exec test -- nslookup kubernetes.default
```

能解析出 IP 说明 CoreDNS 和 Pod 网络正常。

---

## 九、集群重置与清理（需要重装时执行）

**所有节点执行：**

```bash
# 1. 重置 kubeadm（会清理 etcd、证书、静态 Pod 清单）
kubeadm reset -f

# 2. 清理 CNI 网络配置与网卡（重要，否则重装后 IP 冲突）
rm -rf /etc/cni/net.d /opt/cni/bin
ip link delete cni0 2>/dev/null
ip link delete flannel.1 2>/dev/null

# 3. 清理 iptables/ipvs 规则
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
ipvsadm --clear 2>/dev/null

# 4. 清理 kubeconfig 与数据
rm -rf $HOME/.kube /var/lib/kubelet /var/lib/dockershim /var/run/kubernetes

# 5. 重启 containerd 和 kubelet
systemctl restart containerd
systemctl restart kubelet
```

---

## 十、生产环境网络端口要求

学习环境可以关闭防火墙，**生产环境请按需放行**：

| 协议 | 端口 | 源 | 目标 | 用途 |
|------|------|-----|------|------|
| TCP | 6443 | Worker / 外部 | Master | Kubernetes API Server |
| TCP | 2379-2380 | Master | Master | etcd 客户端/对等通信 |
| TCP | 10250 | Master | 所有节点 | Kubelet API |
| TCP | 10259 | Worker | Master | kube-scheduler |
| TCP | 10257 | Worker | Master | kube-controller-manager |
| TCP | 10256 | Worker | Worker | kube-proxy（healthz/metrics） |
| UDP | 8472 | 所有节点 | 所有节点 | Flannel VXLAN  overlay 网络 |
| UDP | 4789 | 所有节点 | 所有节点 | Calico VXLAN 模式 |
| TCP | 30000-32767 | 外部 | Worker | NodePort 服务范围 |

---

## 十一、常见问题排查速查

### Q1: `kubectl get nodes` 显示 NotReady

```bash
# 查看节点事件
kubectl describe node <节点名>

# 常见原因：
# 1. CNI 没装 → 安装 Flannel/Calico
# 2. kubelet 起不来 → journalctl -u kubelet -f
# 3. containerd 没启动 → systemctl status containerd
# 4. 时间不同步 → chronyc tracking
```

### Q2: `kubeadm init` 拉镜像超时

```bash
# 先单独拉取验证
kubeadm config images pull --image-repository=registry.aliyuncs.com/google_containers

# 如果还是失败，手动 docker/ctr 拉取后重新打 tag
ctr -n k8s.io image pull registry.aliyuncs.com/google_containers/kube-apiserver:v1.28.2
```

### Q3: Worker join 失败，提示 token 过期

```bash
# Master 上重新生成
kubeadm token create --print-join-command
```

### Q4: Pod 无法访问 Service / 跨节点不通

```bash
# 检查内核模块
lsmod | grep br_netfilter

# 检查 sysctl
sysctl net.bridge.bridge-nf-call-iptables

# 检查 CNI 插件是否存在
ls /opt/cni/bin/ | grep bridge
```

### Q5: kubelet 状态显示不断重启（安装后、init 前）

**这是正常的**。kubelet 在加入集群前没有 `/var/lib/kubelet/config.yaml`，会反复退出。执行 `kubeadm init` 或 `join` 后自动生成配置，状态即恢复正常。

---

## 附录：一键检查清单

在 Master 上执行前，逐条确认：

- [ ] 3 台节点互通（`ping` 验证）
- [ ] Swap 已关闭（`free -h` 确认 swap 为 0）
- [ ] SELinux 已关闭（`getenforce` 返回 Disabled）
- [ ] 时间已同步（`date` 3 台一致，chronyd 运行中）
- [ ] 内核模块已加载（`lsmod | grep overlay` / `br_netfilter`）
- [ ] sysctl 已生效（`sysctl net.ipv4.ip_forward` 返回 1）
- [ ] containerd 运行中且配置正确（`systemctl status containerd`，`cat /etc/containerd/config.toml | grep SystemdCgroup` 为 true）
- [ ] pause 镜像已预拉取（`ctr -n k8s.io image ls | grep pause`）
- [ ] kubeadm/kubelet/kubectl 版本一致（`kubeadm version` / `kubelet --version` / `kubectl version`）
- [ ] Master 初始化成功且 kubectl 可用（`kubectl get nodes`）
- [ ] CNI 已安装且 Pod 全 Running（`kubectl get pods -A`）
- [ ] Worker join 成功且节点 Ready
