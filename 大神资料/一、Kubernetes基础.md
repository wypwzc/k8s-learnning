### 1. 为什么要用Kubernetes
在业务开始进行容器化时，前期需要容器化的项目可能并不多，涉及的容器也并不多，此时基于Docker容器直接部署至宿主机也能实现自己的需求。但是随着项目越来越多，管理的容器也越来越多，此时使用“裸容器”部署的方式管理起来就显得很吃力，并且随着业务量的增加，会明显体会到“裸容器”的不足，比如：

+ 宿主机宕机造成该宿主机上的容器不可用，且无法自动恢复。
+ 容器明明在运行，接口就是不通（健康检查做得不到位）。
+ 应用程序部署、回滚、扩缩容困难。
+ 成百上千的容器和涉及的端口难以维护。

> Tips：虽然有docker-compose、docker-swarm这些编排工具，但功能还是少，小规模还可以。
>



**开发人员、测试人员定位业务问题**

传统场景：由于公司业务多，开发环境、测试环境、预生产环境和生产环境都是隔离的，而且除了生产环境，为了节省成本，其他环境可能没有进行日志收集。在没有用Kubernetes的时候，查看线下测试的日志，**需要开发或者测试人员找到对应的机器，再找到对应的容器**，才能查看日志。



容器编排场景：在使用Kubernetes之后，开发和测试人员直接在Kubernetes的Dashboard上**找到对应的Namespace，即可定位到业务的容器**，然后可以**直接通过控制台查看到对应的日志**，大大降低了操作时间。



**运维人员繁琐工作问题**

对于运维人员，传统架构可能需要装系统、装依赖环境、部署域名、开通权限等，这一整套下来，不仅耗时，而且可能会因为有某些遗漏而造成诸多问题。

传统场景：在传统架构体系下，公司业务故障可能是因为基础环境不一致、依赖不一致、端口冲突等问题。

容器编排场景：使用Docker镜像部署，Kubernetes进行编排，所有的依赖、基础都是一样的，并且环境的自动化扩容、健康检查、容灾、恢复都是全自动的，大大减少了因为这类基础问题引发的故障。



### 2. Kubernetes架构解析
#### 2.1 架构图
<img src="https://tupian-1251747426.cos.ap-guangzhou.myqcloud.com/image-20230712125032643.png" width="965" alt="" title="" crop="0,0,1,1" id="k1X0d" class="ne-image">



#### 2.2 Kubernetes架构
+  **主（Master）节点** 
    - 其中主节点为集群的控制单元，负责核心的调度、管理和运维，一般不会运行业务应用程序。
    - 主要包含的组件有Kube-APIServer、Kube-ControllerManager、Kube-Scheduler。
+  **从（Worker/Node）节点** 
    - Slave节点负责控制由 K8S 创建的容器的启动停止，保证节点工作正常。
    - 主要包含的组件有Kubelet、Kube-Proxy。
+  **数据库etcd** 
    - 作为保存 Kubernetes 所有集群数据的后台数据库。
    - 负责保存Kubernetes Cluster的配置信息和各种资源的状态信息。
    - 当数据发生变化时，etcd 会快速地通知Kubernetes相关组件。
+  **高可用、负载均衡** 
    - HAproxy、Keepalived实现高可用负载均衡
    - HAproxy前端配置一个访问端口（例如访问VIP+端口号），负载到后端配置集群各主机（各Master的IP+k8s API端口号，例如默认的6443）
    - Keepalived多Master节点上进行配置权重，优先级等，最后再配置一个检测脚本，检查间隔、降权、失败重试次数等等。



**架构实现小结**

一个集群中可以有很多Node节点，用以保证集群容器的分布式部署用于实现业务的高可用性，也可以有很多Master节点。

通过一个负载均衡保证集群控制节点的高可用。负载均衡可以使用软件负载均衡Nginx/LVS/HAProxy+KeepAlived或者硬件负载均衡F5等，通过负载均衡对Kube-APIServer提供的VIP即可实现Master节点的高可用。其他组件通过该VIP连接至Kube-APIServer。

etcd集群可以和Master节点部署在同一个宿主机，也可以单独部署，生产环境建议部署大于3的奇数台etcd节点实现etcd集群的高可用。



#### 2.3 Master节点组件
##### 2.3.1 ectd
ETCD：它是兼具一致性和高可用性的键值key-value数据库，可以作为保存 Kubernetes 所有集群数据的后台数据库，负责保存Kubernetes Cluster的配置信息和各种资源的状态信息，当数据发生变化时，etcd 会快速地通知Kubernetes相关组件。

Kubernetes 集群的 etcd 数据库通常需要有个备份计划。此外还有一种k8s集群部署的高可用方案是将etcd数据库从容器中抽离出来，单独作为一个高可用数据库部署，从而为k8s提供稳定可靠的高可用数据库存储。

> Tips：生成环境如果机器配置够高，安装etcd与master安装在一起也OK，也可以单独分离。
>



##### 2.3.2 kube-apiserver
k8s集群的控制平面的核心是API服务器，而API服务器主要就是由kube-apiserver组件实现的，它被设计为可水平扩展，即通过部署不同数量的实例来进行缩放从而适应不同的流量。API服务器作为整个k8s控制平面的前端，负责提供 HTTP API，以供用户、集群中的不同部分组件和集群外部组件相互通信。（可以通过kubeadm或者kubectl这类的CLI工具来操控API从而控制整个k8s集群，也可以通过其他的Web UI来进行操控）

> Tips：工具在背后也是调用 API，包括Web操作。
>



##### 2.3.3 kube-scheduler
主节点上的组件，该组件监视那些新创建的未指定运行节点的 Pod，并选择最佳节点让 Pod 在上面运行。

调度决策考虑的因素包括单个 Pod 和 Pod 集合的资源需求、硬件/软件/策略约束、亲和性和反亲和性规范、数据位置、工作负载间的干扰和最后时限。

Scheduler将资源供应与工作负载需求相匹配以维持系统的稳定和可靠性，因此Scheduler在调度的过程中需要考虑公平、资源高效利用、效率等方面的问题。



##### 2.3.4 kube-controller-manager
K8S所有Worker Node的监控器。Controller Manager作为集群内部的管理控制中心，运行在Master节点上，是一个永不休止的循环。它保证Pod或其他资源达到期望值。当集群中某个Pod的副本数或其他资源因故障和错误导致无法正常运行，没有达到设定的值时，Controller Manager会及时发现并执行自动化修复流程，确保集群始终处于预期的工作状态。

它实质上是群内的Node、Pod、服务（Server）、端点（Endpoint）、命名空间（Namespace）、服务账号（ServiceAccount）、资源定额（ResourceQuota）的监视与守护进程的合集，每个资源的Controller通过API Server提供的接口实时监控每个资源对象的当前状态。

其中控制器包括:

+ 节点控制器（Node Controller）: 负责在节点出现故障时进行通知和响应。
+ 副本控制器（Replication Controller）: 负责为系统中的每个副本控制器对象维护正确数量的 Pod。
+ 端点控制器（Endpoints Controller）: 填充端点(Endpoints)对象(即加入 Service 与 Pod)。
+ 服务帐户和令牌控制器（Service Account & Token Controllers）: 为新的命名空间创建默认帐户和 API 访问令牌。



Scheduler和Controller Manager虽然部署了多个节点，但同时工作的节点只有一个，因为Scheduler和Controller Manager属于有状态服务，为了防止重复调度，多个节点的Scheduler和Controller Manager进行了选主工作，工作节点（主节点）信息保存在Scheduler和Controller Manager的EndPoint中。

```bash
[root@k8s-master01 ~]# kubectl get leases -n kube-system
NAME                      HOLDER                                              AGE
kube-controller-manager   k8s-master02_d616d87d-fe01-4735-ae1a-40e30c583dbe   16h
kube-scheduler            k8s-master02_40c6225b-d340-4fc5-8032-27c47df3f473   16h
```



#### 2.4 Node节点组件
Node节点也被称为Worker、Node，是主要负责部署容器（工作负载）的单机，集群中的每个节点都必须具备容器的Runtime（运行时），比如Docker或其他遵循CRI标准的Runtime等。



##### 2.4.1 kubelet
K8S中Master节点在每个Worker Node节点上运行的主要“节点代理”，也可以说是Master 的“眼线”。它会定期向Master Node汇报自己Node上运行服务的状态，并接受来自Master Node的指示采取调整措施。负责控制由 K8S 创建的容器的启动停止，保证节点工作正常。当Node节点宕机或故障（NotReady状态）时，该节点上运行的Pod会被自动转移到其他节点上。



##### 2.4.2 kube-proxy
kube-proxy是集群中每个节点上运行的网络代理，负责Node在K8S的网络通讯、以及对外网络流量的负载均衡。 kube-proxy通过维护主机上的网络规则并执行连接转发，实现了Kubernetes服务抽象。

service在逻辑上代表了后端的多个Pod，外界通过service访问Pod。service接收到的请求就是通过kube-proxy转发到Pod上的，kube-proxy服务负责将访问service的TCP/UDP数据流转发到后端的容器。**如果有多个副本，kube-proxy会实现负载均衡。**

由于性能问题，目前大部分企业用K8S进行实际生产时，都不会直接使用Kube-proxy作为服务代理，而是通过Ingress Controller来集成HAProxy、Nginx来代替Kube-proxy。



#### 2.5 其余插件
##### 2.5.1 CoreDNS
CoreDNS：用于Kubernetes集群内部Service的解析，可以让Pod把Service名称解析成Service的IP，然后通过Service的IP地址连接到对应的应用上。



##### 2.5.2 Calico
Calico：符合CNI标准的一个网络插件，它负责给每个Pod分配一个不会重复的IP，并且把每个节点当作一个“路由器”，这样一个节点的Pod就可以通过IP地址访问其他节点的Pod。



##### 2.5.3 Metrics
Metrics：在Kubernetes中是用于监控和度量集群、节点、容器和其他资源的数据。通过Metrics，您可以获取容器级别的指标，例如容器的CPU使用率、内存使用率、网络流量等。这有助于您监控和管理容器的性能，并进行容器级别的故障排除和优化。



##### 2.5.4 Dashboard
Dashboard：是一个基于Web的用户界面，用于管理和监控Kubernetes集群。它提供了一个直观和可视化的方式，让用户能够查看和操作集群中的资源、工作负载、节点和应用程序。

