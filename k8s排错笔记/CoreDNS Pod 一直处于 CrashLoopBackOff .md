## **CoreDNS Pod 一直处于 `CrashLoopBackOff` 状态**，无法 `Ready`。

### 一、错误现象

- **CoreDNS Pod 一直处于 `CrashLoopBackOff` 状态**，无法 `Ready`。
- `kubectl get nodes` 显示节点 `NotReady`（后来修复 Flannel 后变为 Ready，但最初不是）。
- master 节点上 `ping 10.244.0.3`（CoreDNS Pod IP）完全丢包。
- 检查 Flannel 发现 `flannel.1` 和 `cni0` 接口状态为 `DOWN`，路由表中没有 `10.244` 相关条目。
- Flannel 日志反复报错：`network is down`，`failed to add vxlanRoute`。

### 二、根本原因（您操作中的核心错误）

**错误点：在使用 `kubeadm init` 初始化集群时，没有指定 `--pod-network-cidr` 参数，导致后续安装 Flannel 网络插件时，集群的 Pod 网络地址段与 Flannel 的默认配置可能不匹配，或者 Flannel 未能正确初始化宿主机的网络接口。**

> 虽然 Kubernetes 默认会为节点分配 `10.244.0.0/16` 的子网（实际观察节点有 `10.244.0.0/24`），但 **显式指定网段是一个最佳实践**。缺失这个参数，结合环境中可能存在 NetworkManager、SELinux 等干扰因素，使得 Flannel 容器在 master 节点上创建 VXLAN 设备 `flannel.1` 后，无法自动将该接口状态设置为 `UP`，也无法添加路由规则，最终导致整个 Pod 网络瘫痪，CoreDNS 因无法连接 API Server 而不断崩溃。

### 三、改正方法（最终有效的解决方案）

您的解决方案是正确的：**彻底重置集群，重新初始化时指定 Pod 网络 CIDR，再安装 Flannel**。

#### 具体步骤（已验证有效）：

1. **所有节点执行彻底清理**：

   ```
kubeadm reset -f
   rm -rf /etc/cni/net.d/ /var/lib/cni/ /run/flannel/ /etc/kubernetes/
   iptables -F && iptables -t nat -F
   systemctl restart containerd kubelet
   ```
   
   

2. **在 master 节点重新初始化（关键：添加 `--pod-network-cidr`）**：

   ```
kubeadm init --pod-network-cidr=10.244.0.0/16 --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers
   ```
   
   

3. **配置 kubectl**：

   ```
mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```
   
   

4. **安装 Flannel 网络插件**（使用国内镜像加速）：

   ```
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
   ```
   
   

5. **等待节点变为 `Ready`**，然后让 worker 节点使用 `kubeadm join` 加入。

### 四、预防建议（避免再次发生）

- **初始化集群时务必指定 `--pod-network-cidr`**，该值必须与后续安装的网络插件所需网段一致（Flannel 默认要求 `10.244.0.0/16`）。

- **如果宿主机使用 NetworkManager**，创建配置文件禁用管理 flannel 接口：

  bash

  ```
  cat > /etc/NetworkManager/conf.d/99-flannel.conf <<EOF
  [keyfile]
  unmanaged-devices=interface-name:flannel.*
  EOF
  systemctl restart NetworkManager
  ```

  

- **如果 SELinux 为 Enforcing 且导致问题**，可临时关闭测试（`setenforce 0`），确认后编写策略或保持关闭（生产环境谨慎）。

- **始终检查节点上 `flannel.1` 和 `cni0` 接口是否为 `UP`**，以及路由表是否存在 `10.244.x.0/24` 条目。

### 五、总结

> **您的错误就是 `kubeadm init` 时漏掉了 `--pod-network-cidr=10.244.0.0/16`，并且没有意识到环境中的 NetworkManager 或 SELinux 会干扰 Flannel 自动抬起网络接口。通过重置集群并正确指定 CIDR，所有问题得到解决。**