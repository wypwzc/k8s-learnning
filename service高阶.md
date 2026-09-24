同学你好。既然你不要概念性复述，想要直击 Linux 内核网络栈的本质，那我们就直接剥离 Kubernetes 的声明式抽象，把 **Service 还原为 Linux 内核中的 Netfilter 规则、IPVS 转发表、Conntrack 状态机以及 DNS 记录**。

以下是作为一名深耕内核与网络栈的架构师，为你梳理的 K8s Service 全链路深度拆解。

## 一、 Service 类型全对比与内核视角访问路径

从内核角度来看，Service 的本质是一组**动态生成的四层流量调度规则**。不同的 Service 类型决定了这些规则挂载在 Netfilter 的哪些 Hook 点，以及暴露的范围。

### 1. 核心类型全对比

| **Service 类型** | **适用场景**                                                 | **边界限制与缺点**                                           | **内核本质/实现机制**                                        |
| ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **ClusterIP**    | 集群内部 Pod 间互访（默认选型）                              | 仅在集群内部节点与 Pod 内可达，外部无法直接路由              | 在所有节点上挂载虚拟 IP（VIP），通过 Netfilter/IPVS 拦截并做 DNAT |
| **NodePort**     | 简单的集群外入站流量引入，适用于开发测试或边缘节点           | 占用宿主机全局端口（默认 30000-32767），大规模部署易端口耗尽；存在双重 NAT 损耗 | 在宿主机所有网卡上监听指定物理端口，将流量导入 `KUBE-NODEPORTS` 链再分发 |
| **LoadBalancer** | 云厂商托管环境下的外网服务暴露                               | 严重依赖云厂商云盘控制器（CCM）；每创建一服务需买一个外部 LB，成本高 | 继承 NodePort 逻辑，利用云厂商 API 自动配置外部负载均衡器，将流量打到 NodePort 上 |
| **ExternalName** | 集群内 Pod 访问集群外的特定服务（如外部 RDS 数据库）         | 仅在七层/应用层通过 DNS 别名（CNAME）重定向，不支持四层 IP 层的代理与负载均衡 | **纯 DNS 级别改写**。CoreDNS 收到请求后返回 CNAME 记录，不生成任何内核网络规则 |
| **Headless**     | 有状态服务（如 StatefulSet/Kafka/K8s 算力集群），需知晓后端具体 Pod 拓扑 | 客户端必须具备识别多 A 记录并自行实现负载均衡/重试的能力     | **无 VIP**。不分配 ClusterIP，CoreDNS 直接将 Service 域名解析到后端所有 Pod IP 的 A 记录列表 |

### 2. 三大视角下的网络访问路径

#### ① 集群内 Pod 访问视角（Pod A -> Service ClusterIP）

- **路径**：Pod A 发出数据包 $\rightarrow$ `veth0` $\rightarrow$ 宿主机网桥（如 `cni0` 或 OVS） $\rightarrow$ 宿主机内核网络栈 $\rightarrow$ 触发 Netfilter 规则（DNAT 改写目标 IP 为 Pod B IP） $\rightarrow$ 路由查找 $\rightarrow$ 目标 Pod B。
- **特征**：数据包不出宿主机（若 Pod 在同节点）或通过 CNI 隧道（VXLAN/Geneve）/三层路由（BGP）跨节点传输，源 IP 为 Pod A 的 IP。

#### ② 集群外用户访问视角（External -> NodePort / LoadBalancer）

- **路径**：外部客户端 $\rightarrow$ 物理网络交换机 $\rightarrow$ 宿主机物理网卡（如 `eth0`） $\rightarrow$ 内核 `PREROUTING` 链 $\rightarrow$ 进入 `KUBE-NODEPORTS` 链 $\rightarrow$ **第一阶段 DNAT**（将目标宿主机 IP+NodePort 改写为后端 Pod IP+TargetPort） $\rightarrow$ **第二阶段 SNAT**（若 `externalTrafficPolicy: Cluster`，会将源 IP 改写为当前节点的内网 IP，防止跨节点回包路由丢失） $\rightarrow$ 跨节点发往目标 Pod。
- **特征**：默认情况下源 IP 会丢失（被 SNAT 改写），可通过设置 `externalTrafficPolicy: Local` 保留源 IP（但流量只能落到当前节点的 Pod，若当前节点无 Pod 则直接丢包）。

#### ③ 宿主机自身访问视角（Host -> ClusterIP）

- **路径**：宿主机本地进程（如 kubelet/本地 O&M 脚本） $\rightarrow$ 内核网络栈发出数据包 $\rightarrow$ 触发 Netfilter 的 `OUTPUT` 链 $\rightarrow$ 进入 `KUBE-SERVICES` 链 $\rightarrow$ 发生 DNAT 改写 $\rightarrow$ 路由查找发往目标 Pod。
- **特征**：不经过 `PREROUTING` 链，直接在 `OUTPUT` 阶段完成 NAT 转换。

## 二、 kube-proxy 三种模式演进与内核本质

kube-proxy 并不是一个真正处理数据包的代理进程，而是一个**控制面组件**，它监听 API Server 的 Service 和 Endpoint/EndpointSlice 资源变动，然后通过内核驱动将这些变动写入内核转发面。

### 1. userspace 模式（已废弃）

- **内核机制**：kube-proxy 在用户态监听一个随机端口。通过 iptables 的 `REDIRECT` 规则，将原本发往 ClusterIP 的流量重定向到该用户态端口。
- **致命缺陷**：每个数据包都需要经历：`网卡 -> 内核态 (Netfilter) -> 用户态 (kube-proxy进程) -> 内核态 (Socket发送) -> 网卡`。两次上下文切换（Context Switch）和多次内存拷贝，在吞吐量大时 CPU 极易因上下文切换被榨干。

### 2. iptables 模式（当前主流默认模式之一）

基于 Linux Netfilter 框架的 `iptables-restore` 批量刷新规则。

#### ① `-A KUBE-SERVICES` 链的 DNAT 规则生成逻辑

所有进出集群的流量首先会进入 `PREROUTING` 和 `OUTPUT` 链，然后跳转到 `KUBE-SERVICES` 链。

一条典型的 ClusterIP 规则生成逻辑如下：

1. 在 `KUBE-SERVICES` 链中匹配目标 IP 是否为 Service 的 ClusterIP。
2. 如果匹配成功，跳转到该 Service 专属的 `KUBE-SVC-XXXX` 链。
3. `KUBE-SVC-XXXX` 链中包含了多条指向具体后端 Pod 的 `KUBE-SEP-XXXX`（Service Endpoint）链。

#### ② 随机负载均衡算法（probability 模块）

iptables 是顺序匹配的，为了实现负载均衡，它利用了 Netfilter 的 `statistic` 模块（带 `--mode random --probability` 参数）。

假设一个 Service 后面有 $n$ 个 Pod，为了确保每个 Pod 被选中的概率都是 $\frac{1}{n}$，规则链的概率计算公式为：

$$P_i = \frac{1}{n - i + 1}$$

其中 $i$ 表示当前处于第几条 Endpoint 规则（从 1 开始）。

例如，后端有 3 个 Pod ($n=3$)：

- 第 1 条规则的概率：$P_1 = \frac{1}{3 - 1 + 1} = \frac{1}{3} \approx 0.3333$
- 第 2 条规则的概率：$P_2 = \frac{1}{3 - 2 + 1} = \frac{1}{2} = 0.5$（在剩下的 2/3 流量中命中 50%，即总流量的 1/3）
- 第 3 条规则（最后一条）：$P_3 = \frac{1}{3 - 3 + 1} = 1.0$（剩下的流量 100% 命中）

#### ③ Conntrack（连接跟踪）状态机的作用

iptables 规则只负责 **SYN 包（建立连接的首包）** 的 DNAT 决策。当首包经过 Netfilter 时，Linux 内核的 Conntrack 模块会在内存中记录一个五元组状态表。

该连接后续的 `ACK`/`DATA` 包直接由内核 Conntrack 状态机自动匹配并执行相同的 NAT 转换，**不再遍历 iptables 规则链**，从而提升了后续包的转发效率。

#### ④ 性能瓶颈：$O(n)$ 级链遍历与 $O(n^2)$ 级规则刷新

- **匹配瓶颈**：iptables 链在内核中是以**链表（Linear List）** 形式存储的。当集群中 Service 和 Pod 数量达到数万级别时，数据包需要盲目遍历成千上万条规则，匹配复杂度为 $O(n)$。
- **刷新瓶颈**：iptables 不支持增量更新。即使只改动一个 Endpoint，kube-proxy 也必须通过 `iptables-restore` 将全量规则从用户态打包传给内核进行重载。在此期间，会产生全局内核锁，在大规模集群中引发严重的网络抖动甚至假死。

### 3. ipvs 模式（大规模集群生产首选）

为了解决 iptables 的 $O(n)$ 瓶颈，K8s 引入了基于 Linux 虚拟服务器（LVS/IPVS）的模式。IPVS 运行在内核态，底层基于**哈希表（Hash Table）**，规则匹配与查找的时间复杂度为 $O(1)$。

#### ① LVS 调度算法

kube-proxy 支持通过 `--ipvs-scheduler` 参数配置多种内核级调度算法：

- `rr` (Round-Robin): 轮询。
- `wrr` (Weighted Round-Robin): 加权轮询。
- `lc` (Least-Connection): 最小连接数（优先分配给当前活跃连接数最少的后端）。
- `sh` (Source Hashing): 源地址哈希，用于实现基于客户端 IP 的会话保持。
- `dh` / `lh`: 目标地址哈希 / 本地节点优先调度。

#### ② `conn_reuse` 模式与 UDP 会话保持

- **conn_reuse 冲突**：在 Linux 内核中，IPVS 默认开启 `net.ipv4.vs.conn_reuse_mode`。当客户端高并发复用相同源端口（如短连接释放后瞬间重用）时，IPVS 会倾向于复用 Conntrack 中已有的旧连接条目。在 K8s 中，这会导致大量新连接被错误地发送到正在销毁的旧 Pod 上，引发连接重置（RST）。生产环境通常需要优化内核参数将该值设为 `0`。
- **UDP 会话保持**：由于 UDP 是无状态的，IPVS 内部维护了一个临时的 UDP 定时器（通常为 5 分钟）。在这个窗口内，相同五元组的 UDP 包会强行锁定在同一个后端 Pod，这在 CoreDNS 扩容时往往会导致流量分配极度不均。

#### ③ 与 iptables 的协作关系

IPVS 只负责四层负载均衡与 NAT，它**无法处理端口伪装（Masquerade）、包过滤（Filter）及各种高级路由标记**。

因此，在 ipvs 模式下，kube-proxy 依然会创建少量的 iptables 规则（利用 `ipset` 匹配 IP 集合），用于实现：

- `KUBE-MARK-MASQ`：对需要做 SNAT 的包打上 `0x4000/0x4000` 标记。
- 在 `POSTROUTING` 链中，对带该标记的流量执行 `MASQUERADE`（伪装出站）。

### 4. nftables 模式（K8s 1.29+ 最新演进）

nftables 是 Linux 内核中旨在完全替代 iptables 的新一代包过滤框架。

- **架构本质**：nftables 引入了一个**轻量级的内核虚拟机（VM）**。它将用户态编写的规则编译成字节码（Bytecode），然后在内核的虚拟机中执行。
- **为什么快**：
  1. **原生支持 Map/Set 结构**：nftables 可以在一条规则里通过集合（Sets）或字典（Maps）完成多目标的联动匹配。查找性能同样是 $O(1)$，完美取代了 `iptables` + `ipset` 的别扭组合。
  2. **原子化增量更新**：支持通过 Netlink 现成套接字进行单条规则的增量插入/删除，无需像 iptables 那样整体重载，彻底解决了大集群中刷新规则导致的内核锁死问题。

## 三、 数据包流转实战路径深度剖析

### 1. 案例一：集群内 Pod 访问 ClusterIP（iptables 模式）

**环境假设**：

- Pod A (源): `10.244.1.5`
- Service ClusterIP: `10.96.0.10:80`
- Pod B (目标): `10.244.2.3:8080`

```
 [Pod A (10.244.1.5)] 
        │ (1. 发出原始包: src=10.244.1.5, dst=10.96.0.10:80)
        ▼
   [veth0 / cni0] ────► [Host Netfilter]
                             │
                             ├─► [PREROUTING] ──► 跳转至 KUBE-SERVICES
                             │                         │
                             │                         ▼ (2. 匹配 ClusterIP)
                             │                  [KUBE-SVC-XXXX] 
                             │                         │
                             │                         ▼ (3. 概率匹配统计模块)
                             │                  [KUBE-SEP-YYYY]
                             │                         │
                             │                         ▼ (4. 执行 DNAT 转换)
                             │                  改写为 dst=10.244.2.3:8080
                             ▼
                    [路由查找 & CNI 封装] ──► (5. 发往跨节点 Pod B)
```

#### 内核链逐跳流转：

1. **Pod A 发包**：构造 TCP 握手包（SYN），`src=10.244.1.5:43210`，`dst=10.96.0.10:80`。数据包通过 `veth` 对弹射到宿主机的 CNI 网桥。
2. **进入宿主机 PREROUTING**：数据包由网桥进入宿主机网络栈，触发 Netfilter `PREROUTING` 链，该链挂载了 `KUBE-SERVICES` 子链。
3. **匹配 ClusterIP 并跳转**：内核检查规则，发现目标地址匹配 `10.96.0.10`，将其引导入 `KUBE-SVC-XXXX` 链。
4. **统计模块选定后端**：在 `KUBE-SVC-XXXX` 链中，内核应用 `statistic` 随机概率，决定命中 `KUBE-SEP-YYYY`（对应后端的 Pod B）。
5. **执行 DNAT**：在 `KUBE-SEP-YYYY` 链中触发 `-j DNAT --to-destination 10.244.2.3:8080`。内核修改 IP 报头，此时包变为 `src=10.244.1.5:43210`，`dst=10.244.2.3:8080`。
6. **路由与发出**：内核根据新的目标 IP `10.244.2.3` 重新查找路由表，通过 CNI 插件的隧道或路由机制，将包发送给 Pod B 所在的宿主机，最终注入 Pod B 的网络空间。

### 2. 案例二：外部用户访问 NodePort（跨节点漂移）

**环境假设**：

- 外部客户端 IP: `202.108.22.3`
- Node 1 IP: `192.168.1.10` (用户请求打在此节点，但此节点无 Pod)
- Node 2 IP: `192.168.1.11` (Pod B 实际驻留在此节点: `10.244.2.3:8080`)
- Service NodePort: `32000`
- `externalTrafficPolicy`: `Cluster`（默认）

#### 数据包流转路径：

1. **入站 Node 1**：包到达 Node 1 的物理网卡，`src=202.108.22.3`，`dst=1192.168.1.10:32000`。
2. **触发 KUBE-NODEPORTS**：在 `PREROUTING` 中未匹配到 ClusterIP，但流转到 `KUBE-NODEPORTS` 链时，成功匹配到目标端口 `32000`。
3. **执行 DNAT**：规则链根据随机概率，选中了驻留在 Node 2 的 `KUBE-SEP-YYYY`。执行 DNAT，目标地址改写为 `10.244.2.3:8080`。
4. **触发 SNAT (MASQUERADE)**：由于数据包的目标 IP 被改成了 Pod 纯内网 IP，且需要**跨节点**发送给 Node 2。如果不做特殊处理，Node 2 上的 Pod B 收到请求后，回包时会直接尝试通过默认网关发送给外部客户端 `202.108.22.3`。
   - **后果**：外部客户端会收到一个来自 `192.168.1.11` (Node 2) 的包，但客户端明明是在和 `192.168.1.10` (Node 1) 握手！客户端内核会直接丢弃该包（发送 RST）。
   - **解决机制**：Node 1 上的 `KUBE-NODEPORTS` 会强制对该包打上 `MASQUERADE` 标记。在 `POSTROUTING` 阶段，内核将源 IP 改写为 Node 1 的内网 IP `192.168.1.10`。
5. **跨节点路由**：此时数据包变为：`src=192.168.1.10`，`dst=10.244.2.3:8080`。顺利通过 CNI 发送到 Node 2 上的 Pod B。

### 3. 为什么回包必须走原路径？（Conntrack 逆向NAT转换）

当后端 Pod B 回包时（`src=10.244.2.3:8080`，`dst=192.168.1.10`）：

1. 包首先回到 Node 1。
2. Node 1 的内核网络栈在收到包后，第一步**不看路由表，而是先过 Conntrack 状态机**。
3. Conntrack 发现该包的四元组反转后，完美匹配之前记录的入站连接追踪条目。
4. **自动执行反向 NAT（Un-NAT）**：内核自动将源 IP 从 Pod B IP 恢复为 Node 1 的物理 IP，将目的 IP 从 Node 1 内网 IP 恢复为外部客户端 IP。
5. 最终出网包变为：`src=192.168.1.10:32000`，`dst=202.108.22.3`。客户端正常接收，TCP 连接建立成功。

### 4. 内核规则查看示例

#### ① 查看 iptables NAT 转换规则：

Bash

```
$ iptables -t nat -L KUBE-SERVICES -n -v
# 输出示例：
# pkts bytes target     prot opt in     out     source               destination
#    10   600 KUBE-SVC-LB75TCP  tcp  --  * * 0.0.0.0/0            10.96.0.10           /* default/nginx-service:http */ tcp dpt:80

$ iptables -t nat -L KUBE-SVC-LB75TCP -n -v
# 展现随机负载均衡机制：
# 0.0.0.0/0            0.0.0.0/0            statistic mode random probability 0.5000000000000 statistic mode random... -> 跳转到第一个SEP
```

#### ② 查看 IPVS 转发表：

Bash

```
$ ipvsadm -Ln
# 输出示例：
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.0.10:80 rr
  -> 10.244.1.5:8080              Masq    1      0          0         
  -> 10.244.2.3:8080              Masq    1      0          0         
# 可以直观看到 ClusterIP 作为虚拟服务器，后端挂载了两个真实 Pod 实例 (Masq模式)
```

#### ③ 查看 Conntrack 实时状态：

Bash

```
$ conntrack -L -p tcp
# 输出示例：
tcp      6 118 ESTABLISHED src=10.244.1.5 dst=10.96.0.10 sport=43210 dport=80 src=10.244.2.3 dst=10.244.1.5 sport=8080 dport=43210 [ASSURED] mark=0 use=1
# 注意：一条追踪记录里包含了“正向 tuple”和期望的“反向 reply tuple”
```

## 四、 DNS 与 Service 发现机制

### 1. CoreDNS 插件解析原理

CoreDNS 通过 `kubernetes` 插件实时向 API Server 订阅 Service 和 Endpoint。

- **ClusterIP 解析**：当查询 `my-svc.my-ns.svc.cluster.local` 的 `A` 记录时，CoreDNS 直接从内存索引中返回该 Service 的 ClusterIP。
- **Headless 解析**：如果是 Headless Service，CoreDNS 内部**没有 ClusterIP 可填**，它会直接查找该 Service 背后所有就绪（Ready）的 Endpoint Pod IP，并打包成一组 `A` 记录同时返回给客户端。
- **Pod DNS 解析**：K8s 会为每个 Pod 自动生成域名。格式为 `pod-ip-with-dash.namespace.pod.cluster.local`（例如 `10-244-1-5.my-ns.pod.cluster.local`），直接解析为该 Pod IP。

### 2. SRV 记录与 ndots 性能陷阱

- **SRV 记录**：用于发现命名的服务端口。例如查询 `_http._tcp.my-svc.my-ns.svc.cluster.local`，会返回该端口对应的具体数值以及目标域名。

- **ndots:5 性能深渊**：

  在 K8s 的 `/etc/resolv.conf` 中，默认配置了 `options ndots:5`。

  它的含义是：**如果请求的域名中包含的“.”的数量少于 5 个，操作系统会优先把这个域名当作相对域名，依次拼接 `search` 域进行迭代查询**。

  - **悲剧案例**：你在 Pod 内请求外部域名 `www.baidu.com`（包含 2 个点 < 5）。
  - **真实发生的查询序列**：
    1. `www.baidu.com.default.svc.cluster.local.` $\rightarrow$ NXDOMAIN (不存在)
    2. `www.baidu.com.svc.cluster.local.` $\rightarrow$ NXDOMAIN
    3. `www.baidu.com.cluster.local.` $\rightarrow$ NXDOMAIN
    4. `www.baidu.com.external-search-domain.` $\rightarrow$ NXDOMAIN
    5. `www.baidu.com.` $\rightarrow$ **成功返回**
  - **影响**：一次简单的外网请求，在线上放大了 **4 倍** 的 DNS 内部无效无效查询，极易把 CoreDNS 打挂。
  - **解法**：高频使用的外网域名，在编写代码请求时在末尾强行加上绝对域名标记 `.`（如 `www.baidu.com.`），或者在 Pod 编排中显式将 `ndots` 调整为 `2`。

### 3. EndpointSlice 分片与拓扑路由

- **EndpointSlice 机制**：传统的 `Endpoints` 资源将一个 Service 下的所有 Pod IP 塞在一个对象里。若有 1000 个 Pod，哪怕一个 Pod 变动，整个大对象就会触发全量 ETCD 读写和所有 Node 的广播通知。`EndpointSlice` 将其切片化（默认每个 Slice 最多放 100个 Endpoint），变动时只更新单个小 Slice，极大减轻了控制面压力。
- **拓扑感知路由（Topology Aware Hints）**：通过给 EndpointSlice 打上 `hints` 标签（如指定当前 Pod 属于 `zone-a`），Node 上的 kube-proxy 在生成 iptables/IPVS 规则时，会优先把流量导向跟当前节点在同一个可用区（Zone）的后端 Pod，实现就近访问，避免跨机房/跨可用区流量带来的延迟和带宽成本。

## 五、 生产环境排障、内核调优与性能实战

### 1. `nf_conntrack: table full` 根因与扩容

- **现象**：业务高并发时，集群内开始出现随机丢包、连接超时，内核日志（`dmesg -T`）爆出：`nf_conntrack: table full, dropping packet`。

- **根因**：Linux 内核用哈希表记录连接状态。如果短连接并发极高，或者存在大量死连接未释放，导致当前存活的 Conntrack 条目数超过了 `net.netfilter.nf_conntrack_max` 阈值，内核为了自保，会直接拒绝为新到的首包（SYN）分配状态，直接执行丢弃（Drop）。

- **内核调优方案**：

  Bash

  ```
  # 查看当前连接跟踪数与最大值
  sysctl net.netfilter.nf_conntrack_count
  sysctl net.netfilter.nf_conntrack_max
  
  # 动态调大最大限制（根据物理内存按需调整）
  sysctl -w net.netfilter.nf_conntrack_max=1048576
  
  # 缩短 TCP 处于 ESTABLISHED 状态的非活跃保持时间（默认高达 5 天，建议缩短）
  sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=86400
  ```

### 2. 短连接场景下的源端口耗尽（TIME_WAIT）

- **现象**：Pod 访问 Service 出现 `Cannot assign requested address` 报错。

- **根因**：客户端高频发起短连接，关闭连接后，Socket 按照 TCP 规范进入 `TIME_WAIT` 状态（持续 2 个 MSL，通常为 60 秒）。在此期间，该本地源端口被锁定。Linux 默认的临时端口范围有限，一旦被占满就无法发起新连接。

- **内核调优方案**：

  Bash

  ```
  # 扩大临时端口本地选择范围
  sysctl -w net.ipv4.ip_local_port_range="1024 65535"
  
  # 开启时间戳并允许重用 TIME_WAIT 状态的 Socket 用于新连接
  sysctl -w net.ipv4.tcp_timestamps=1
  sysctl -w net.ipv4.tcp_tw_reuse=1
  ```

### 3. 会话亲和性与 IPVS `sh` 算法的冲突

- **现象**：配置了 `sessionAffinity: ClientIP`，但每当集群扩容后端 Pod 时，原本固定的客户端连接依然会漂移到新 Pod 上，无法做到绝对的持久化。
- **根因**：IPVS 的 `sh`（Source Hashing）算法基于静态一致性哈希。当后端 Pod 列表（桶的大小与内容）发生变更时，哈希桶重新映射，原本的哈希计算结果必定发生改变。
- **生产避坑**：如果业务对长连接/Session 状态有极端严格的绑定需求，四层 Service 不是最佳选择，应上浮至七层 Ingress / Gateway API，利用 Cookie 注入或高级的一致性哈希算法来实现。

### 4. 健康检查与 Endpoint 移除的时序竞赛（优雅停机）

当一个 Pod 被执行删除时，集群内会并发触发两条独立的故事线：

```
                        [ 用户执行 kubectl delete pod ]
                                       │
                ┌──────────────────────┴──────────────────────┐
                ▼                                             ▼
     [ 流程 A: Pod 自身状态机 ]                       [ 流程 B: 控制面感知 ]
                │                                             │
    1. 状态变更为 Terminating                     1. EndpointSlice 移除该 Pod IP
                │                                             │
    2. 执行 preStop Hook                          2. 异步广播至所有节点 kube-proxy
                │                                             │
    3. 发送 SIGTERM 信号                          3. 节点刷新 iptables/IPVS 规则
                │                                             │
    4. 超过 Grace Period -> SIGKILL                           ▼
                ▼                                     [ 停止新流量导入该 Pod ]
        [ 进程彻底退出 ]
```

- **致命死锁/时序冲突**：流程 A 与 流程 B 是**完全异步**的。如果你的应用程序在收到 `SIGTERM` 后立刻停服并退出，此时各个节点的 kube-proxy 可能还没来得及刷新完内核规则！

- **后果**：残留的 iptables 规则依然会将后续的新请求源源不断地送往这个已经死去的进程，从而在线上引发大量的 `502 Bad Gateway`。

- **标准生产解法**：在 Pod 的 `lifecycle.preStop` 中强行注入一个几秒钟的 sleep 延迟，给控制面和全网 kube-proxy 刷新内核规则留出足够的时间差：

  YAML

  ```
  lifecycle:
    preStop:
      exec:
        command: ["/bin/sh", "-c", "sleep 15 && /usr/sbin/nginx -s quit"]
  ```

### 5. 三大真实故障案例及 tcpdump 抓包命令

#### 案例一：间歇性连接抖动，提示 Connection Reset

- **定位排查**：在宿主机物理网卡及网桥上双向抓包，观察是否有大量的 `RST` 包。

- **排查命令**：

  Bash

  ```
  tcpdump -i any -nnvv 'tcp[tcpflags] & (tcp-rst|tcp-syn) != 0' and host <Service_IP>
  ```

- **分析**：若发现大量相同五元组的 SYN 被回应了 RST，极大可能是 IPVS 的 `conn_reuse_mode` 内核问题，依照上文将其关闭即可。

#### 案例二：外部接入流量严重不均衡，部分 Pod 被“撑死”，部分“饿死”

- **定位排查**：检查外部请求是否主要为长连接或特定的 UDP 协议。

- **排查命令**：

  Bash

  ```
  # 实时查看 IPVS 转发计数器，观察入站包大小变化
  watch -n 1 "ipvsadm -Ln --stats"
  ```

- **分析**：如果使用了 `Cluster` 流量策略且经过了多层 NAT，所有流量的源 IP 会收敛为几台固定的前端 LB 节点 IP。此时如果 kube-proxy 配置的是 `sh` 调度，会导致流量全部砸向个别 Pod。应切换为 `lc`（最小连接数）调度。

#### 案例三：K8s 节点间互访丢包，跨节点访问 Service 偶尔超时

- **定位排查**：怀疑是 VXLAN 隧道封装导致的 MTU 超限，导致大包在经过某些交换机时被强行分片或丢弃。

- **排查命令**（发起大包 Ping 测试）：

  Bash

  ```
  # 在 Pod 内向跨节点的目标发起禁止分片的探测大包
  ping -M do -s 1472 <Target_Pod_IP>
  ```

- **分析**：如果常规包通，大包不通且提示 `Frag needed`，说明 CNI 的 MTU 设置过大（例如物理网卡 MTU 是 1500，VXLAN 需要额外 50 字节头部，则 CNI 网卡 MTU 必须设为 1450）。

## 六、 负载均衡的本质：四层转发 vs 七层代理

很多初学者容易把 K8s Service 与 Nginx 混淆。其实它们的内部工作机制有本质不同：

1. **Service 是“伪负载均衡器”（转发模式）**：
   - 无论是 iptables 的统计概率，还是 IPVS 的转发，它们都属于**四层网络交换（Layer 4 Forwarding）**。
   - 内核**不跟客户端三次握手**。它只负责在数据包流经内核网卡时，闭着眼睛改写 IP 报头（NAT），然后直接转发给后端。内核根本不知道、也无法解析 HTTP 协议头、URL 路径或 Cookie。
2. **Ingress / Gateway API 才是真正的负载均衡器（代理模式）**：
   - Ingress（如 Envoy, Nginx Ingress Controller）工作在**七层（Layer 7 Proxy）**。
   - 它是真正的用户态独立进程。外部客户端必须**先和 Ingress 进程彻底建立 TCP 连接（完成三次握手）**。
   - Ingress 解析完整的 HTTP 请求，根据 Host、Path、Header 做出业务层路由决策，随后由 Ingress 进程作为客户端，发起**第二段全新的 TCP 连接**发往后端的 Pod。

## 结论：K8s Service 网络排障速查卡

### 1. 核心诊断命令快查表

| **排查目标**      | **核心诊断命令**                                | **关键关注指标/状态**                      |
| ----------------- | ----------------------------------------------- | ------------------------------------------ |
| **内核连接跟踪**  | `conntrack -S`                                  | `insert_failed`（若不为 0 则代表表满丢包） |
| **IPVS 转发表**   | `ipvsadm -Ln --rate`                            | `CPS`, `InPPS`（检查是否有流量倾斜）       |
| **iptables 命中** | `iptables -t nat -L -n -v | grep -i <svc_name>` | 计数器（pkts/bytes）是否持续递增           |
| **DNS 解析耗时**  | `dig +trace +stats @<CoreDNS_IP> <Domain>`      | `Query time` 以及返回的响应状态            |
| **端口监听状态**  | `ss -antlp | grep <port>`                       | `Send-Q`, `Recv-Q`（是否有队列溢出积压）   |

### 2. 经典异常根因检查清单（Checklist）

#### 🔴 现象一：访问 Service 没有任何响应（直接超时）

- [ ] 检查后端 Pod 的 `ReadinessProbe` 状态。如果健康检查未通过，EndpointSlice 会自动将其剥离，Service 背后无 IP 可投递。
- [ ] 执行 `kubectl get svc -o wide`，确认该 Service 的 ClusterIP 是否分配成功。
- [ ] 在节点执行 `ipvsadm -Ln`，确认该 ClusterIP 对应的真实后端条目（ActiveConn）是否存在。

#### 🟡 现象二：间歇性连接超时 / 偶发性大文件传输中断

- [ ] 查看宿主机内核日志：`dmesg -T | grep -i conntrack`，排查是否由于高并发导致 `nf_conntrack: table full`。
- [ ] 检查各节点网卡 MTU 配置，确认 CNI 容器网卡的 MTU 是否比宿主机物理网卡 MTU 至少小 50 字节（为隧道封装留出空间）。

#### 🔵 现象三：业务日志中获取到的源 IP 全是集群内部节点的内网 IP（源 IP 丢失）

- [ ] 检查该 Service 的 `externalTrafficPolicy` 配置。如果为 `Cluster`，流量发生跨节点漂移时会被强制执行 `MASQUERADE`（SNAT）。
- [ ] **修复对策**：将其改写为 `Local`。但需注意，改写后外部负载均衡器必须具备节点健康检查能力，确保请求只打向真正驻留了 Pod 的节点，否则会导致无 Pod 节点直接丢包。