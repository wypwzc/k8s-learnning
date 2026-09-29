恭喜你！看到 `bootstrap done` 并且监听到其他节点的子网事件，说明你的 Kubernetes 网络插件已经彻底打通。

针对你这次从 `CrashLoopBackOff` 到成功部署的过程，我为你总结了核心的排查逻辑和解决方案，建议收藏，以后扩容或重装时可以参考：

### 1. 核心问题复盘：为什么之前一直失败？

- **API Server 连接闭环（最主要原因）**：Flannel 默认通过 `10.96.0.1`（Service IP）连接 API Server。但在网络插件没跑通前，这个虚拟 IP 在 Master 节点上往往是不可达的。这导致了 `connection refused` 报错。
- **环境残留**：由于多次尝试，`/etc/cni/net.d` 目录下的旧配置或 `flannel.1` 虚拟网卡的残留状态导致了新进程无法初始化。
- **网卡识别问题**：多网卡环境下（如虚拟机），Flannel 有时会找错物理网卡，导致无法跨节点通信。

### 2. 最终生效的关键操作

- **绕过虚拟 IP**：在 `kube-flannel.yml` 的环境变量中手动指定了 `KUBERNETES_SERVICE_HOST` 为物理 IP `192.168.36.137`，打破了“先有网络还是先有插件”的死锁。
- **彻底环境清理**：删除了 `/etc/cni/net.d/*` 和 `/var/lib/cni/*`，并手动清理了 `iptables` 规则，给了 `kube-proxy` 和 `flannel` 一个干净的底层环境。
- **指定物理网卡**：在启动参数中显式添加了 `--iface=ens33`。

### 3. 目前集群的健康标准（请检查确认）

如果现在环境是正常的，你应该能看到以下现象：

1. **节点状态**：执行 `kubectl get nodes`，所有节点状态均为 **`Ready`**。
2. **虚拟网卡**：执行 `ip a show flannel.1`，能看到 IP 地址（通常是 `10.244.0.0` 或 `10.244.0.1`）。
3. **配置文件**：`/etc/cni/net.d/10-flannel.conflist` 已经由 Pod 自动生成或手动补全。
4. **跨节点通信**：在 Master 上 ping 另外两个 Node 的 `flannel.1` IP 应该是通的。

### 4. 维护建议

- **防火墙与 SELinux**：确保 `firewalld` 保持关闭，`SELinux` 保持 `Permissive` 或 `Disabled` 状态。
- **静态 IP**：因为你在配置中写死了 `192.168.36.137`，请务必确保 Master 节点的物理 IP 不会因为 DHCP 重启而改变。
- **镜像备份**：既然 `ghcr.io` 的镜像已经拉取成功，建议不要轻易删除，或者将其 `docker save` 备份一份，防止以后节点扩容时拉不到镜像。

你的 K8s 集群网络现在已经正式开启。接下来你可以放心地部署 Nginx 或其他应用，测试 Pod 之间的网络连通性了！