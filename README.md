# Kubernetes 自学与运维排错笔记

这里记录我学习 Kubernetes 的过程，包括核心概念、集群搭建、网络通信、常用资源实践，以及排查集群故障时的思路和复盘。内容会随着学习和实际操作持续补充。

## 内容导航

### 学习路线与基础概念

- [Kubernetes 学习路线](<k8s自学笔记/0-学习路线.md>)
- [适合我的学习路线](<k8s自学笔记/0-适合我的路线.md>)
- [Kubernetes 架构与核心组件](<k8s自学笔记/！Kubernetes 架构与核心组件.md>)
- [重要概念](<k8s自学笔记/重要概念.md>)

### 网络

- [网络基础概念与核心组件](<k8s网络/1.网络基础概念与核心组件.md>)
- [Pod 内部通信](<k8s网络/2.Pod 内部通信.md>)
- [Pod 与 Pod 通信](<k8s网络/3.Pod 与 Pod 通信.md>)
- [Pod → Service → Ingress 流量链路](<k8s网络/4.Pod → Service → Ingress 流量链路.md>)
- [Node 与 Node 通信](<k8s网络/5.node与node通信.md>)
- [Node 与外网通信](<k8s网络/6.Node 与外网通信.md>)
- [网络阶段总结](<k8s网络/网络阶段总.md>)

### 资源与实战

- [Pod 之间如何通信](<k8s自学笔记/1-pod之间如何通信？.md>)
- [Service](<k8s自学笔记/2-service.md>)、[Ingress](<k8s自学笔记/3-ingress.md>) 与 [Gateway API](<k8s自学笔记/4-gateway.md>)
- [Deployment](<k8s自学笔记/Deployment.md>)、[StatefulSet](<k8s自学笔记/statefulset.md>) 与 [DaemonSet](<k8s自学笔记/Daemonset.md>)
- [ConfigMap 与 Secret 联合实践](<k8s自学笔记/6-ConfigMap + Secret + Deployment 联合实战.md>)
- [PV、PVC 与 NFS](<k8s自学笔记/8-pv-pvc-sc-nfs详解.md>)
- [HPA](<k8s自学笔记/9-HPA.md>) 与 [蓝绿部署](<k8s自学笔记/蓝绿部署.md>)

### 集群搭建与故障排查

- [Kubernetes 集群搭建笔记](<k8s自学笔记/k8s集群的搭建.md>)
- [排错笔记目录](<k8s排错笔记/k8s.md>)
- [三节点集群搭建故障复盘](<k8s排错笔记/K8s 3 节点集群搭建故障复盘与解决方案.md>)
- [CoreDNS Pod 持续 CrashLoopBackOff](<k8s排错笔记/CoreDNS Pod 一直处于 CrashLoopBackOff .md>)
- [Flannel 故障记录](<k8s排错笔记/flannel一直01.md>)
- [镜像拉取问题](<k8s排错笔记/拉取镜像问题.md>) 与 [Pause 容器问题](<k8s排错笔记/pause容器拉不起来.md>)

## 建议阅读顺序

1. 先了解 Kubernetes 架构、核心概念和基础资源。
2. 再按网络章节梳理 Pod、Service、Ingress 之间的流量路径。
3. 结合实战笔记学习配置管理、存储、工作负载和弹性伸缩。
4. 遇到集群问题时，从排错记录中对照现象、检查过程和解决方案。

## 笔记说明

- 这是个人学习笔记和实践记录，命令及配置应结合自己的 Kubernetes 版本、运行时和集群环境确认后再使用。
- 排错记录侧重问题现象、定位过程和处理结果；不同环境下的根因和解决方式可能不同。
- 部分内容仍在整理中，目录和链接会随笔记更新。
