你好！作为一名在 Kubernetes 及底层网络摸爬滚打了 10 年的老兵，我非常乐意为你拆解这套复杂但极具魅力的技术栈。Kubernetes 的网络并非凭空捏造，它完全长在 Linux 内核网络之上。

我们将按照你要求的 8 个阶段，由浅入深、剥茧抽丝地进行系统性推演。

### 第一阶段：Linux 网络基础

**核心概念讲解**

K8s 网络的基础是隔离与连通。网络命名空间（Network Namespace, netns）提供了 L2-L4 的协议栈隔离（独立的网卡、路由表、iptables 规则）。为了打破隔离，我们需要跨 netns 通信的“网线”——**veth pair**（虚拟以太网对，一端发包另一端必收包）。当多个 netns 需要互相通信时，我们需要一个虚拟交换机——**Linux Bridge**。ARP 机制则负责在二层网络中解析 IP 到 MAC 接口的映射。

**组件交互图（文字描述）**

```
[netns:ns1(veth1)] <--> [Bridge(br0)] <--> [(veth2)netns:ns2]
```

**实战：纯手工打造“迷你单节点 K8s 网络”**

```
# 1. 创建 namespace（模拟两个 Pod）
ip netns add ns1
ip netns add ns2

# 2. 创建 Bridge（模拟 docker0 或 cni0）
ip link add br0 type bridge
ip link set br0 up

# 3. 创建 veth pair 并挂载
ip link add veth1 type veth peer name veth1-br
ip link set veth1 netns ns1
ip link set veth1-br master br0 up

ip link add veth2 type veth peer name veth2-br
ip link set veth2 netns ns2
ip link set veth2-br master br0 up

# 4. 配置 IP 并启动
ip -n ns1 addr add 10.0.0.1/24 dev veth1
ip -n ns1 link set veth1 up
ip -n ns2 addr add 10.0.0.2/24 dev veth2
ip -n ns2 link set veth2 up

# 5. 验证测试
ip netns exec ns1 ping -c 2 10.0.0.2
```

**故障排查案例**

- **现象**：`ping` 不通。
- **排查**：首先查 L2 连通性。执行 `ip netns exec ns1 arp -n`，如果处于 `incomplete`，说明 ARP 请求未响应。接着用 `bridge fdb show br br0` 检查 Bridge 是否学到了目标 MAC。通常是因为 `veth*-br` 接口未 UP 导致 ARP 广播被丢弃。

> **阶段递进**：通过 veth 和 Bridge，我们实现了**同宿主机**的网络隔离与互通。但如果是 ns1 要访问公网呢？仅靠 Bridge 的 L2 转发是不够的，我们需要 L3 的路由和 NAT 转换，这就引出了第二阶段的核心：iptables。

### 第二阶段：iptables

**核心概念讲解**

Netfilter 是 Linux 内核中的报文处理框架，iptables 是其用户态工具。数据包在内核中有 5 个关键的 Hook 点（PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING）。

当容器（内部私有 IP）需要访问外网时，必须在出宿主机物理网卡前将源 IP 替换为宿主机的公网 IP（**SNAT / MASQUERADE**）。当外网请求访问 NodePort 时，需要将宿主机 IP:Port 替换为容器 IP:Port（**DNAT**）。

**数据包流经路径**

```
入站包 -> PREROUTING (DNAT?) -> 路由判断 -> (转发) -> FORWARD -> POSTROUTING (SNAT?) -> 出站
```

**实战：使用 iptables 实现容器访问外网**

假设外网网卡是 `eth0`，容器网段是 `10.0.0.0/24`。

Bash

```
# 1. 开启内核 IPv4 转发能力
sysctl -w net.ipv4.ip_forward=1

# 2. 在宿主机网关上设置 NAT 伪装 (SNAT 的动态版)
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE

# 3. 允许从 bridge 转发出去的流量
iptables -t filter -A FORWARD -i br0 -o eth0 -j ACCEPT
iptables -t filter -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# 为 ns1 添加默认路由指向宿主机 bridge 的 IP (假设 br0 设了 10.0.0.254)
ip netns exec ns1 ip route add default via 10.0.0.254
ip netns exec ns1 ping 8.8.8.8
```

**故障排查案例**

- **现象**：能 ping 通宿主机，但 ping 不通 8.8.8.8。
- **排查**：使用 `conntrack -L -s 10.0.0.1` 查看连接跟踪表。如果只有 `[UNREPLIED]` 状态，说明回包路由不对或 SNAT 没生效。用 `iptables -t nat -nvL POSTROUTING` 检查 MASQUERADE 规则的报文匹配计数器（pkts）是否增加。

> **阶段递进**：此时我们具备了网络隔离、同节点互通、跨网段路由和 NAT 转换的基础机制。但在 K8s 中，成千上万个 Pod 频繁生灭，谁来自动化执行这些 `ip netns` 和 `iptables` 命令呢？这正是第三阶段 CNI 要解决的问题。

### 第三阶段：CNI（Container Network Interface）

**核心概念讲解**

CNI 是一套标准规范，kubelet 并不直接操作底层网络，而是通过 JSON 配置文件调用 CNI 插件（二进制可执行文件）。CNI 插件暴露了标准操作：`ADD`（创建 Pod 时分配 IP 并挂载网卡）、`DEL`（删除时清理资源）、`CHECK`。

IPAM（IP Address Management）是 CNI 中的重要子模块，负责 IP 池管理。如 `host-local` 通过本地磁盘文件记录已分配的 IP，防止冲突。

**实战：手写一个极简 CNI 插件 (bash)**

kubelet 会通过环境变量（而非传参）将 Pod 信息传递给 CNI 插件，并通过标准输入（stdin）传入网络配置。

Bash

```
#!/bin/bash
# 最小化 CNI 脚本: minicni
# 依赖 jq 解析 json

# 读取 stdin 的 JSON 配置
config=$(cat)
cniVersion=$(echo $config | jq -r .cniVersion)
subnet=$(echo $config | jq -r .ipam.subnet) # 偷懒假设这里直接拿到 10.0.0.0/24

if [ "$CNI_COMMAND" == "ADD" ]; then
    # 随机分配一个 IP (仅为演示，极其简陋)
    IP="10.0.0.$((RANDOM % 253 + 2))"
    
    # 核心动作：创建 veth，移入 netns
    ip link add $CNI_IFNAME type veth peer name veth-$CNI_CONTAINERID
    ip link set $CNI_IFNAME netns $CNI_NETNS
    ip netns exec $CNI_NETNS ip addr add $IP/24 dev $CNI_IFNAME
    ip netns exec $CNI_NETNS ip link set $CNI_IFNAME up
    
    # 按照 CNI 规范向 stdout 输出分配结果
    cat <<EOF
{
  "cniVersion": "$cniVersion",
  "ips": [
    { "version": "4", "address": "$IP/24" }
  ]
}
EOF
fi
```

**故障排查案例**

- **现象**：Pod 一直处于 `ContainerCreating` 状态。

- **排查**：查看 kubelet 日志 `journalctl -u kubelet -f`，通常会看到 `network plugin is not ready` 或执行 CNI 返回非 0 状态码。此时可模拟 kubelet 手动执行：

  `CNI_COMMAND=ADD CNI_NETNS=/var/run/netns/xxx CNI_IFNAME=eth0 ./minicni < config.json`，看 stdout 输出了什么报错。

> **阶段递进**：CNI 确保了每个 Pod 生来就有一个可路由的独立 IP。但在 K8s 中，Pod IP 极易变化，客户端需要一个固定的入口来访问服务。这引入了 Service 概念，而将 Service 映射到 Pod 的幕后黑手，就是第四阶段的 kube-proxy。

### 第四阶段：kube-proxy

**核心概念讲解**

kube-proxy 是 Node 节点上的网络代理，监听 API Server 中的 Service 和 Endpoint 变化，并据此修改本机的流量转发规则。

1. **userspace**：最早的模式，流量进内核后被导向用户态的 proxy 进程，再发往目标 Pod。上下文切换开销极大，已淘汰。
2. **iptables**：默认模式。完全在内核态执行。利用 iptables 的 `statistic` 模块实现随机负载均衡（如 `mode random probability 0.5`）。依赖 `conntrack` 表维护会话连接。
3. **IPVS**：高性能模式。基于内核 L4 负载均衡，采用 Hash 表存储规则，支持 WRR、LC 等丰富调度算法。

**性能对比与规则流转**

- iptables 模式下，如果有 10000 个 Service，iptables 链会变得无比冗长（O(N) 遍历查找），新建连接首包延迟飙升。

- IPVS 模式下，查找时间复杂度是 O(1)。

- **iptables 规则链剖析**：

  流量命中 `PREROUTING` -> 跳转到自定义链 `KUBE-SERVICES` -> 匹配目标 IP 为 Service IP -> 跳转到 `KUBE-SVC-XXX` -> 按概率跳转到具体 Pod 的 `KUBE-SEP-XXX` (DNAT)。

**故障排查案例**

- **现象**：`curl <ClusterIP>` 偶尔超时，或者修改 Service 端口后旧连接仍然打到旧后端。
- **排查**：这是典型的 `conntrack` 老化慢问题。通过 `conntrack -L | grep <Service-IP>` 查看内核状态表。K8s 中有时需要手动清除旧连接：`conntrack -D -d <Service-IP>` 强制丢弃旧会话记录。

> **阶段递进**：kube-proxy 建立好了底层的转发规则（IPVS 或 iptables）。在第五阶段，我们将以全局视角，把 DNS 发现和底层转发串起来，走一遍 Service 通信的完整生命周期。

### 第五阶段：Service 通信完整剖析

**核心概念讲解**

- **ClusterIP**：集群内虚拟 IP。
- **NodePort**：在宿主机开启端口，供集群外访问（本质是劫持宿主机的特定端口流量走 DNAT）。
- **CoreDNS**：监听 Service 创建，自动生成记录（如 `svc.namespace.svc.cluster.local -> ClusterIP`）。
- **EndpointSlice**：当 Service 后端 Pod 过多时，K8s 1.19+ 引入了 EndpointSlice 替代传统的 Endpoint，避免 API Server 在 Pod 变动时广播巨大的数据对象。

**数据包追踪全过程：Pod A `curl http://svc-b` -> Pod B**

1. **DNS 解析**：Pod A 发起 UDP 请求，经宿主机发往 CoreDNS Pod IP，拿到 svc-b 的 ClusterIP (如 10.96.0.10)。
2. **发包出 Pod**：Pod A 构造目的 IP=10.96.0.10 的报文，经 veth pair 抵达宿主机内核。
3. **命中 Netfilter**：宿主机内核 PREROUTING 链捕获该包，进入 `KUBE-SERVICES` 链。
4. **DNAT 发生**：匹配到 10.96.0.10 后，在 `KUBE-SVC-*` 链中经过负载均衡，决定发给 Pod B (10.244.1.5)。报文目的 IP 在这一刻被**改写为 10.244.1.5**。
5. **路由决策**：内核发现 10.244.1.5 是集群内的合法 Pod IP，按照主机路由表将其发往正确的网络设备（可能是 flannel.1 或者网关）。

**故障排查案例**

- **现象**：通过 Service IP 可访问，但通过域名不可访问。
- **排查**：登入 Pod A 执行 `nslookup svc-b`，如果失败，查 `/etc/resolv.conf` 里的 `nameserver` 是否指向 kube-dns 的 IP。接着查 CoreDNS Pod 是否就绪：`kubectl get pods -n kube-system -l k8s-app=kube-dns`，并查看其日志排除 upstream 报错。

> **阶段递进**：在步骤 5 中，如果 Pod B 刚好在另一台 Node 上，宿主机内核该如何把包投递到远端物理机？这超出了原生 Linux 路由的范畴。由此进入第六阶段：集群网络大动脉——Flannel 与 Calico。

### 第六阶段：Flannel / Calico 对比

**核心概念讲解**

- **Overlay（覆盖网络）**：把 Pod 报文强行塞进宿主机的报文（UDP 或 IP）里传输。屏蔽底层网络差异，但有额外封包解包开销。代表：Flannel VXLAN。
- **Underlay（底层网络）**：不封装包，直接在物理机的路由表里注入 Pod 路由。性能接近物理机，但需要底层网络设备（如 BGP 路由器）配合。代表：Calico BGP 模式。

**Flannel VXLAN 与 Calico BGP 报文形态差异**

- **Flannel VXLAN 抓包**：

  外层 MAC (NodeA->NodeB) | 外层 IP (NodeA->NodeB) | UDP Header | VXLAN Header | **内层 MAC (PodA->PodB veth)** | **内层 IP (PodA->PodB)** | Payload

- **Calico BGP 抓包**：

  外层 MAC (NodeA->NodeB) | **目标 IP 已经是 PodB** | Payload

  *(注意：Calico BGP 没有内层 MAC 和额外的 Header，它纯粹依靠 NodeB 用自身 MAC 代答目标 IP，然后 NodeB 本地的三层路由将其丢入目标 Pod veth)*

**Flannel vs Calico vs Cilium（前瞻）横向对比**

| **特性**             | **Flannel (VXLAN)**      | **Calico (BGP)**       | **Cilium (eBPF)**              |
| -------------------- | ------------------------ | ---------------------- | ------------------------------ |
| **网络模型**         | Overlay (MAC-in-UDP)     | Underlay (纯 L3 路由)  | 支持纯路由机制 / Overlay       |
| **数据路径**         | 内核协议栈 + netdev      | 内核纯 L3 路由转发     | eBPF 直接在网卡/驱动层拦截     |
| **网络策略(Policy)** | 不支持                   | 支持 (基于 iptables)   | 强 (L3-L7，基于 eBPF Map)      |
| **适用场景**         | 中小型集群，不挑底层网络 | 大型集群，对性能要求高 | 超大集群，微服务安全，极致性能 |

**故障排查案例**

- **现象**：Calico 集群跨节点 Pod 无法通信。
- **排查**：在源宿主机执行 `ip route` 检查是否包含目标 Pod 网段下一跳（指向目标 Node IP）。执行 `calicoctl node status` 检查 BGP 邻居连接是否为 `Established`。如果不是，通常是 Node 之间 179 端口被防火墙阻断。

> **阶段递进**：我们对比了 Overlay 和 Underlay。虽然 Calico 性能更好，但在云原生环境（如公有云 VPC 内不允许私自运行 BGP），基于 VXLAN 的 Overlay 仍是绝对主流。我们将在第七阶段扒开 VXLAN 的黑盒。

### 第七阶段：深入剖析 VXLAN

**核心概念讲解**

VXLAN（Virtual eXtensible Local Area Network）是一种 NVO3 网络虚拟化技术。

- **VTEP (VXLAN Tunnel End Point)**：隧道端点（在 Flannel 中就是 `flannel.1` 这个设备），负责封包和解包。
- **FDB (Forwarding Database)**：转发表。VTEP 怎么知道内层目标 MAC 地址在哪个远程 Node 的 VTEP 后面？全靠 FDB 记录。
- **ARP 抑制 (ARP Suppression)**：如果全网广播 ARP 去查内部 MAC 对应的宿主机 IP，会引发网络风暴。Linux 内核支持代理 ARP 响应，Flannel daemon 会把所有 Node 的 MAC 和 IP 映射写入本机的 ARP 表和 FDB 表中，从而抑制广播。

**实战：纯手工搭建跨节点 VXLAN 隧道**

假设 Node1(192.168.1.10) 和 Node2(192.168.1.20)。

Bash

```
# 在 Node1 上执行：
# 1. 创建 VTEP 设备 vxlan0 (VNI=100, 基于 UDP 4789)
ip link add vxlan0 type vxlan id 100 dstport 4789 local 192.168.1.10
ip link set vxlan0 up
# 2. 为隧道网卡分配内部 IP
ip addr add 10.200.0.1/24 dev vxlan0
# 3. 手动向 FDB 注入规则：发往全 0 MAC 的报文，封装后扔给对端宿主机 192.168.1.20
bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 192.168.1.20

# 在 Node2 上对称执行配置 (local=192.168.1.20, vtep IP=10.200.0.2, fdb dst 192.168.1.10)
# (此处省略 Node2 代码)

# 验证测试 (在 Node1 上):
ping 10.200.0.2
```

**故障排查案例**

- **现象**：Flannel 跨节点不通，抓包发现有外层 UDP(4789) 发出，对端却未收到内层 ICMP 请求。
- **排查**：使用 `bridge fdb show dev flannel.1` 确认转发表是否有目标 VTEP 的 MAC 地址及其对应的宿主机 IP。如果是云环境（如 AWS/阿里云），极大概率是云控制台的**安全组（Security Group）未开放 UDP 4789 端口**。

> **阶段递进**：VXLAN 解决了底层隔离环境下的互通问题，但代价是极长的内核协议栈穿透（封包解包、无数次 netfilter hook）。网络架构师们开始思考：能不能绕过内核 TCP/IP 协议栈？这就是最终阶段的杀手锏——eBPF 与 Cilium。

### 第八阶段：eBPF 与 Cilium 架构

**核心概念讲解**

- **eBPF**：Linux 内核提供的沙盒虚拟机，允许在内核中安全、极速地运行自定义代码，而无需修改内核源码。
- **XDP (eXpress Data Path)**：eBPF 的一个重要 Hook 点，**位于网卡驱动层，比内核协议栈还要靠前！** 报文刚到网卡就能被丢弃或重定向。
- **TC (Traffic Control)**：另一个 Hook 点，适合做网络策略控制和封装/解封装。
- **Map 机制**：用户态和内核态 eBPF 程序高效共享数据的内存键值存储。
- **Cilium 架构**：通过 Cilium Agent 监听 K8s API，实时生成 BPF C 字节码并注入内核（网卡的 TC/XDP 钩子上），配合 Hubble 提供基于 Identity 的 L7 流量图谱观测。

**为什么 eBPF 性能远超 iptables？（Maglev 与负载均衡）**

Cilium 开启 kube-proxy replacement 后：

1. 无需 iptables 的 O(N) 链表遍历，直接在 eBPF Map 中执行 O(1) 的 Hash 查找获取 Service 的后端 Endpoint。
2. 数据包无需进入宿主机 IP 路由、netfilter 框架，直接在 TC 甚至 XDP 钩子上完成 DNAT。
3. **短路机制 (Short-circuit)**：同节点 Pod 通信时，报文离开 PodA 的 veth，直接被 eBPF 重定向 (bpf_redirect) 塞进 PodB 的 veth，完全绕开宿主机协议栈，犹如瞬间移动！

**核心方案与组件对比总结**

| **纬度**                | **iptables (Kube-Proxy)** | **IPVS (Kube-Proxy)** | **eBPF (Cilium)**                   |
| ----------------------- | ------------------------- | --------------------- | ----------------------------------- |
| **规则匹配复杂度**      | O(N) 线性遍历             | O(1) 哈希表           | O(1) eBPF Maps                      |
| **数据包处理点**        | 协议栈 Netfilter 钩子     | 协议栈 Netfilter 钩子 | **网卡驱动 (XDP) / TC 层** (极早期) |
| **上下文切换/内核遍历** | 极长（多次穿透内核层）    | 较长                  | **极短**（短路转发）                |
| **可观测性**            | 黑盒，依赖 conntrack      | 黑盒                  | 原生支持 (Hubble, L3-L7 协议级追踪) |

**故障排查案例**

- **现象**：启用 Cilium 后，某个应用访问特定服务一直 Drop。
- **排查**：由于报文在进入 tcpdump 能够抓包的阶段前可能就被 XDP 丢弃了，传统工具失效。必须使用 Cilium 自己的工具：执行 `hubble observe --pod <pod-name> --verdict DROPPED`，直接看到包是被哪个 NetworkPolicy (Identity ID) 在内核哪一层拦截掉的。也可以使用 `bpftool map dump id <map-id>` 窥探底层路由/策略 Map 的真实数据。

至此，我们从最底层的 veth 和 Bridge 起步，经过 iptables 的打磨，利用 CNI 将其工程化，用 kube-proxy 编排服务网络，用 Flannel/Calico 连通多节点，最后用 eBPF(Cilium) 完成了性能与可观测性的降维打击。这就是现代云原生网络架构演进的完整脉络。如果你对其中某一层（比如如何手写 BPF 代码替换 kube-proxy）感兴趣，我们可以进一步深入探讨。