## 二、初识Pod
### 1. 什么是Pod
Pod是工作负载在 Kubernetes 上运行的应用程序。在 Kubernetes 中，Pod代表的是集群上处于运行状态的**一组 容器的集合**。



**Pod的设计初衷**

如果把某个应用的进程进行拆分，拆分成一个一个容器，那就有可能出现某个进程容器被调度到了不同的节点上，往往应用内部的进程与进程间通信（通过 IPC 或者共享本地文件之类）都是要求在本地进行的，也就是需要在同一个节点上运行。

所以需要一个更高级别的结构来将这些容器绑定在一起，并将他们作为一个基本的调度单元进行管理，这样就可以保证这些容器始终在同一个节点上面，这也就是 Pod 设计的初衷。

<img src="https://tupian-1251747426.cos.ap-guangzhou.myqcloud.com/image-20230717215336707.png" width="452" alt="" title="" crop="0,0,1,1" id="TGNWU" class="ne-image">

Pod可简单地理解为一组、一个或多个容器，每个Pod还包含一个Pause容器，**Pause容器是Pod的父容器，它主要负责僵尸进程的回收管理，同时通过Pause容器可以使同一个Pod里面的不同容器共享存储、网络、PID、IPC等**，容器之间可以使用localhost:port相互访问，可以使用Volume等实现数据共享。

> 根据Docker的构造，Pod可被建模为一组具有共享命名空间、卷、IP地址和端口的容器。
>



### 2. 引入Pod的原因
轻量级：Pod 是 Kubernetes 集群中最小的可部署单元，可以轻松地部署、调度和管理。它们可以根据需要创建、销毁或重启，而无需影响集群中的其他部分。

资源共享：Pod 中的容器共享相同的网络和存储资源。这使得它们可以相互通信并访问共享的文件系统。例如，多个容器可以共享同一份配置文件，而不必在每个容器中复制。

网络隔离：Pod 提供了一个虚拟网络环境，可以为容器提供独立的 IP 地址和网络命名空间。这使得容器可以相互隔离，并且可以防止容器之间的网络干扰。

服务发现：Pod 可以被标记和注释，以便 Kubernetes 可以根据标签和注释选择它们。这使得容器可以轻松地与其他容器通信，并允许 Kubernetes 自动发现和管理服务之间的关系。

水平扩展：通过使用 Pod 副本集，可以轻松地创建多个相同配置的 Pod，并在需要时自动扩展它们。这使得应用程序可以根据负载自动调整其资源使用率，并提高可扩展性和可用性。



**在之前Docker方案时，就会有如下的情况发生：**

使用裸容器时，需要将容器内应用程序的端口映射到宿主机，如果容器过多，端口管理就会比较困难，而且容易引起端口冲突。



**使用Kubernetes方案解决：**

而Kubernetes为每个Pod都分配一个唯一的IP地址，这样就可以保证不同应用程序可以使用同一个端口，之后通过Kubernetes的内部Service进行访问，这样就避免了发生端口冲突的问题。



### 3. 了解Pod原理
其实 Pod 也只是一个逻辑概念，真正起作用的还是 Linux 容器的 Namespace 和 Cgroup 这两个最基本的概念，Pod 被创建出来其实是一组共享了一些资源的容器而已。

**网络**

+ 首先 Pod 里面的所有容器，都是共享的同一个 Network Namespace。

**存储**

+ 但是涉及到文件系统的时候，默认情况下 Pod 里面的容器之间的文件系统是完全隔离的，但是我们可以通过声明来共享同一个 Volume，也就是挂载同一存储卷**（默认情况下，在一个Pod中的容器A无法直接查看或访问容器B的文件系统，它们之间的文件操作是隔离的）**。



在Kubernetes中，一个Pod可以包含多个容器，这些容器共享同一个网络命名空间（Network Namespace）、存储卷和其他资源。尽管这些容器在Pod内部运行，但在底层的Docker引擎中，它们仍然会以多个独立的容器的形式存在，都有自己的Docker运行时实例，并拥有独立的容器ID。



一个示例，找一个Pod，确认在哪个节点运行（图中为k8s-node02），接下来到node02节点上查看docker运行的容器，证明一个Pod下多个容器都有自己的Docker运行时实例，并拥有独立的容器ID。

<img src="https://tupian-1251747426.cos.ap-guangzhou.myqcloud.com/image-20230717233038073.png" width="1325" alt="" title="" crop="0,0,1,1" id="xLNJ8" class="ne-image">

<img src="https://tupian-1251747426.cos.ap-guangzhou.myqcloud.com/image-20230717233354626.png" width="1348" alt="" title="" crop="0,0,1,1" id="u1RYL" class="ne-image">

那么容器的启动有先后顺序问题，那么 Pod 是怎么来处理这个问题的呢？那就是加入一个中间容器pause。这个容器在 Pod 中永远都是第一个被创建的容器，这样是不是其他容器都加入到这个pause容器就可以了，这样就完全实现了 Pod 中的所有容器都和pause容器共享同一个 Network Namespace 了。

> 使用pause，这也是为什么在 Kubernetes 里面，它是允许去单独更新 Pod 里的某一个镜像的，即：做这个操作，整个 Pod 不会重建，也不会重启，这是非常重要的一个设计。
>



普通容器不会创建自己的网卡，配置自己的 IP，而是和pause容器共享 IP、端口范围等，而且容器之间的进程可以通过 lo 网卡设备进行通信：

+ 也就是容器之间是可以直接使用 localhost 进行通信的；
+ 看到的网络设备信息都是和 pause 容器完全一样的；
+ 也就意味着同一个 Pod 下面的容器运行的多个进程不能绑定相同的端口；
+ 而且 Pod 的生命周期只跟pause容器一致，而与容器 A 和 B 无关。



Kubernetes通过使用网络代理和IP表来实现Pod内的通信，并将对Pod的请求路由给相应的容器。这使得在Pod内的多个容器能够以高效的方式共享资源、相互通信和协作工作。



### 4. 了解init容器
Init 容器是 Kubernetes 中的一种特殊容器，用于在主容器启动之前运行特定的初始化任务。它们通常用于执行一次性的预配置或数据初始化操作，以确保主容器的正常启动和运行。**（Pod 的 YAML 文件可以不指定 Init 容器。Init 容器是可选的）**

一些常见的用途包括：

1.  配置和准备环境：Init 容器可以在主容器启动之前设置和准备环境。例如，可以使用 Init 容器来加载配置文件、初始化数据库、创建目录结构等。 
2.  同步和等待：在某些情况下，主容器可能需要依赖其他组件或服务。Init 容器可以用于等待这些依赖项可用后再启动主容器。例如，可以使用 Init 容器等待数据库或消息队列可用后再启动应用程序容器。 
3.  数据预加载：有时候，主容器需要访问大量数据或文件，而这些数据可能需要提前加载到共享卷中。Init 容器可用于在主容器启动之前将数据从外部存储源（如对象存储或数据库）复制到共享卷中。 

通过使用 Init 容器，可以确保主容器在正确的环境和依赖项下运行，从而提高应用程序的可靠性和稳定性。在 Kubernetes 中，**Init 容器与主容器共享相同的网络和存储空间，并且它们会按照定义的顺序依次执行。只有当所有的 Init 容器都成功完成后，主容器才会被启动。**

> Tips：如果 Pod 的 Init 容器失败，Kubernetes 会不断地重启该 Pod，直到 Init 容器成功为止。然而，如果 Pod 对应的 `restartPolicy` 为 Never，它不会重新启动。
>



例子如下：

当使用 Init 容器时的一个例子是在部署一个 Web 应用程序时，需要进行一些预处理任务，例如安装依赖项或执行数据库迁移。以下是一个示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-web-app
spec:
  containers:
    - name: main-app
      image: my-app-image
      # 主应用程序容器配置
      # ...
  initContainers:
    - name: init-setup
      image: busybox
      command: ["sh", "-c", "echo 'Initializing setup...'"]
      # 初始化设置容器配置
      # ...
    - name: init-database
      image: mysql:5.7
      env:
        - name: MYSQL_ROOT_PASSWORD
          value: mysecretpassword
      command: ["sh", "-c", "mysql -uroot -p$MYSQL_ROOT_PASSWORD -e 'CREATE DATABASE mydb;'"]
      # 数据库初始化容器配置
      # ...
```

这个示例中，我们定义了一个 Pod 包含一个主应用程序容器 `main-app` 和两个 Init 容器 `init-setup` 和 `init-database`。

`main-app` 是运行 Web 应用程序的主容器，我们假设它是由 `my-app-image` 镜像构建的。

`init-setup` 是一个 Init 容器，它使用 `busybox` 镜像，并通过 `echo 'Initializing setup...'` 命令打印出初始化设置的信息。

`init-database` 是另一个 Init 容器，使用 `mysql:5.7` 镜像，通过环境变量 `MYSQL_ROOT_PASSWORD` 设置 MySQL root 密码，并使用 `mysql -uroot -p$MYSQL_ROOT_PASSWORD -e 'CREATE DATABASE mydb;'` 命令创建名为 `mydb` 的数据库。

这样，当 Pod 启动时，它将按照定义的顺序依次执行这两个 Init 容器。首先，`init-setup` 容器将打印初始化设置的信息。然后，`init-database` 容器将初始化数据库。

通过定义这些 Init 容器，我们可以在主应用程序启动之前进行一些预处理任务，以确保应用程序正常运行。你可以根据需求添加或修改 Init 容器的配置。



### 5. Pod状态
Pod在运行时也会有不同的状态，Pod的状态信息保存在PodStatus对象中，在PodStatus中有一个Phase字段，用于描述Pod在其生命周期中的不同状态。

```bash
kubectl get pod
```

<img src="https://tupian-1251747426.cos.ap-guangzhou.myqcloud.com/image-20230718133111984.png" width="689" alt="" title="" crop="0,0,1,1" id="sEjKE" class="ne-image">

```bash
#kubectl get pod <Podname> -oyaml
kubectl get pod cluster-test-8b47d69f5-dbvvf -oyaml
```

<img src="https://tupian-1251747426.cos.ap-guangzhou.myqcloud.com/image-20230718133453917.png" width="248" alt="" title="" crop="0,0,1,1" id="XT04u" class="ne-image">

> Tips：kube-controller-manager就是用来控制 Pod 的状态和生命周期的
>



#### 5.1 Pending（挂起）
Pod 信息已经提交给了集群，但仍有一个或多个容器未被创建。可以通过`kubectl describe pod -n 命名空间名称 Pod名称` 查看处于 Pending 状态的原因。



#### 5.2 Running（运行中）
Pod已经被绑定到一个节点上，并且所有的容器都已经被创建，而且至少有一个是运行状态、正在启动或者重启，可以通过 `kubectl logs -n 命名空间名称 Pod名称` ，查看Pod的日志。

> Tips：Pod状态为Running，并不代表Pod正常，也需要READY为N/N
>



#### 5.3 Succeeded（成功）
所有容器执行成功并终止，并且不会再次重启。可以通过`kubectl logs -n 命名空间名称 Pod名称` 查看Pod日志。



#### 5.4 Failed（失败）
所有容器都已终止，并且至少有一个容器以失败的方式终止，也就是说这个容器要么以非零状态退出，要么被系统终止。

可以通过`kubectl logs -n 命名空间名称 Pod名称`、`kubectl describe pod -n 命名空间名称 Pod名称`查看Pod日志和状态原因。



#### 5.5 Unknown（未知）
通常是由于通信问题造成的无法获得 Pod 的状态。



#### 5.6 ImagePullBackOff ErrImagePull
镜像拉取失败，一般是由于镜像不存在、网络不通或者需要登录认证引起的。

可以通过`kubectl describe pod -n 命名空间名称 Pod名称`查看Pod状态原因。





#### 5.7 CrashLoopBackOff
容器启动失败，可以通过 logs 命令查看具体原因，一般为启动命令不正确，健康检查不通过，前台没有该进程等。

可以通过 `kubectl logs -n 命名空间名称 Pod名称` 查看Pod日志。



#### 5.8 OOMKilled
容器内存溢出，一般是容器的内存 Limit 设置的过小，或者程序本身有内存溢出。

可以通过 `kubectl logs -n 命名空间名称 Pod名称` 查看Pod日志。



#### 5.9 Terminating
Pod 正在被删除。可以通过`kubectl describe pod -n 命名空间名称 Pod名称`查看Pod状态原因。



#### 5.10 SysctlForbidden
Pod 自定义了内核配置，但 kubelet 没有添加内核配置或配置的内核参数不支持。

可以通过`kubectl describe pod -n 命名空间名称 Pod名称`查看Pod状态原因。



#### 5.11 Completed
容器内部主进程退出，一般计划任务执行结束会显示该状态。

可以通过 `kubectl logs -n 命名空间名称 Pod名称` 查看Pod日志。



#### 5.12 ContainerCreating
Pod 正在创建，一般为正在下载镜像，或者有配置不当的地方。

可以通过`kubectl describe pod -n 命名空间名称 Pod名称`查看Pod状态原因。



### 6. 定义一个Pod
编写yml文件

```yaml
[root@k8s-master01 ~]# cat /yaml/pod/test-nginx.yaml
apiVersion: v1 #必选，API 的版本号
kind: Pod      #必选，类型 Pod
metadata:      #必选，元数据
  name: nginx  #必选，Pod名称
spec: #必选，用于定义 Pod 的详细信息
  containers:  #必选，容器列表
  - name: test-nginx #必选，容器名称
    image: nginx:1.14.2  #必选，容器镜像地址
    ports:  #可选，容器需要暴露的端口号列表
    - containerPort: 80  #暴露的端口号
```



创建Pod

```bash
#yaml两种方式
cd /yaml/pod/
kubectl create -f test-nginx.yaml

kubectl create -f /yaml/pod/test-nginx.yaml

#也可以直接run一个pod
kubectl run nginx-run --image=nginx:1.15.12
```



查看Pod的状态

```bash
[root@k8s-master01 pod]# kubectl get pod
NAME                           READY   STATUS              RESTARTS       AGE
cluster-test-8b47d69f5-dbvvf   1/1     Running             21 (18m ago)   39h
nginx                          0/1     ContainerCreating   0              11s

[root@k8s-master01 pod]# kubectl get pod -n  default -owide
NAME                           READY   STATUS    RESTARTS       AGE   IP              NODE         NOMINATED NODE   READINESS GATES
cluster-test-8b47d69f5-dbvvf   1/1     Running   21 (30m ago)   39h   172.16.58.199   k8s-node02   <none>           <none>
nginx                          1/1     Running   0              12m   172.16.85.205   k8s-node01   <none>           <none>
```



### 7. Pod的镜像拉取策略
当你最初创建一个 Deployment、 StatefulSet、Pod 或者其他包含 Pod 模板的对象时，如果没有显式设定的话， Pod 中所有容器的默认镜像拉取策略是 `IfNotPresent`（除非镜像 tag 为 latest 或没有指定标签）。这一策略会使得 kubelet在镜像已经存在的情况下直接略过拉取镜像的操作。

| 操作方式 | 说明 |
| --- | --- |
| Always | 总是拉取，当镜像 tag 为 latest 或没有指定标签时，且 imagePullPolicy 未配置，默认为 Always |
| Never | 不管是否存在都不会拉取，如果镜像已经以某种方式存在本地， kubelet 会尝试启动容器；否则，会启动失败。 |
| IfNotPresent | 镜像不存在时拉取镜像，如果 tag 为非 latest，且 imagePullPolicy 未配置，默认为 IfNotPresent。 |




yaml示例如下，可通过`kubectl describe pod <Pod名称>`查看效果。

```yaml
apiVersion: v1 #必选，API 的版本号
kind: Pod      #必选，类型 Pod
metadata:      #必选，元数据
  name: nginx  #必选，Pod名称
spec: #必选，用于定义 Pod 的详细信息
  containers:  #必选，容器列表
  - name: test-nginx #必选，容器名称
    image: nginx:1.14.2  #必选，容器镜像地址
    ports:  #可选，容器需要暴露的端口号列表
    - containerPort: 80  #暴露的端口号
    imagePullPolicy: Always #总是拉取
```

> Tips：Pod镜像拉取策略是针对Pod中单个容器的行为。
>



### 8. Pod的重启策略
Pod 的 `spec` 中包含一个 `restartPolicy` 字段，其可能取值包括 Always、OnFailure 和 Never。默认值是 Always。

| 操作方式 | 说明 |
| --- | --- |
| Always | 默认策略，容器失效时，自动重启该容器。 |
| OnFailure | 容器以不为 0 的状态码终止，自动重启该容器。 |
| Never | 无论何种状态，都不会重启。 |




yaml示例如下，可以通过在节点上进行`docker stop 容器名称`来看策略效果，具体效果查看通过`kubectl get pod`。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
    command: ["sleep","3"]
  restartPolicy: OnFailure
```

> Tips：Pod重启策略是针对Pod里的所有容器。
>



### 9. Pod的三种探针
在生产环境下，进程正常启动并不代表应用能正常处理请求，所以合理地设计应用的健康检查尤为重要。在使用裸机或者裸容器部署时，一般很难对应用做很完善的健康检查，而Pod提供的探针可以很方便地用来检测容器内的应用是否正常。目前探针有3种检测方式，可以根据不同的场景选择合适的健康检查方式。



#### 9.1 探针种类、实现形式说明
**Pod探针的种类**

在Kubernetes中，可以通过两种类型的探针来监测Pod中的容器：

1. 存活探针（Liveness Probe）：用于监测容器是否仍然处于运行状态。如果存活探针检测到容器失败，Kubernetes将会自动重启该容器。
2. 就绪探针（Readiness Probe）：用于监测容器是否已准备好接收流量。如果就绪探针检测到容器无法处理流量，Kubernetes将会停止将流量发送到该容器，直到就绪探针重新检测到容器可接收流量。

| 种类 | 说明 |
| --- | --- |
| startupProbe（启动探针） | Kubernetes1.16 新加的探测方式，用于判断容器内的应用程序是否已经启动。如果 配置了startupProbe，就会先禁用其他探测，直到它成功为止。如果探测失败，Kubelet 会杀死容器，之后根据重启策略进行处理，如果探测成功，或没有配置 startupProbe， 则状态为成功，之后就不再探测。 |
| livenessProbe（存活探针） | 用于探测容器是否在运行，如果探测失败，kubelet 会“杀死”容器并根据重启策略进行相应的处理。如果未指定该探针，将默认为 Success。 |
| readinessProbe（就绪探针） | 一般用于探测容器内的程序是否健康，即判断容器是否为就绪（Ready）状态。如果是，则可以处理请求，反之 Endpoints Controller 将从所有的 Service 的 Endpoints 中删除此容器所在Pod的IP地址。如果未指定，将默认为Success。 |




**Pod探针的实现方式**

| 实现方式 | 说明 |
| --- | --- |
| Exec | 在容器内执行一个指定的命令，如果命令返回值为0，则认为容器健康 |
| TCPSocket | 通过TCP连接检查容器指定的端口，如果端口开放，则认为容器健康 |
| HTTPGet（最可靠） | 对指定的URL进行Get请求，如果状态码在200~400(不包括400) 之间，则认为容器健康 |
| gRPC | 1.24版本开始出现，使用gRPC执行一个远程过程调用。 目标应该实现gRPC健康检查。 如果响应的状态是 "SERVING"，则认为诊断成功。 gRPC 探针是一个 Alpha 特性，只有在你启用了 "GRPCContainerProbe" 特性门控时才能使用。 |




**探针结果**

| 结果 | 说明 |
| --- | --- |
| Success | 容器通过检测 |
| Failure | 容器未通过检测 |
| Unknown | 检测失败，因此不会采取任何措施。 |




#### 9.2 创建一个无探针Pod
创建一个没有探针的 Pod

```yaml
[root@k8s-master01 pod]# cat /yaml/pod/pod.yaml
apiVersion: v1 # 必选，API 的版本号
kind: Pod # 必选，类型 Pod
metadata: # 必选，元数据
    name: nginx # 必选，符合 RFC 1035 规范的 Pod 名称
spec: # 必选，用于定义 Pod 的详细信息
    containers: # 必选，容器列表
    - name: nginx # 必选，符合 RFC 1035 规范的容器名称
      image: nginx:1.15.12 # 必选，容器所用的镜像的地址
      imagePullPolicy: IfNotPresent
      command: # 可选，容器启动执行的命令
      - sh
      - -c
      - sleep 20; nginx -g "daemon off;"
      ports: # 可选，容器需要暴露的端口号列表
      - containerPort: 80 # 端口号
    restartPolicy: Never  #重启策略，无论如何都不会重启
```



创建Pod，并查看pod分配的IP，最后再curl一下IP。此过程就是演示创建Pod后，由于已有镜像，此时Pod已经Running，也分配了IP，此时Service会把这个Pod IP添加进去到Endpoints 列表。但此时进行curl IP却说端口没通，那么如果此时有流量负载到这个Pod，那就会丢包。**（只有Pod的Ready为N/N时，Service才会分配流量到该Pod）**

```bash
[root@k8s-master01 pod]# kubectl create -f pod.yaml
pod/nginx created

[root@k8s-master01 pod]# kubectl get pod -owide
NAME    READY   STATUS    RESTARTS   AGE   IP              NODE         NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          3s    172.16.58.213   k8s-node02   <none>           <none>

[root@k8s-master01 pod]# curl 172.16.58.213
curl: (7) Failed connect to 172.16.58.213:80; Connection refused

#yaml写的sleep后再启动nginx进程，Running 20秒后再curl就能访问到nginx了。
[root@k8s-master01 pod]# kubectl get pod -owide
NAME    READY   STATUS    RESTARTS   AGE   IP              NODE         NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          21s   172.16.58.213   k8s-node02   <none>           <none>

[root@k8s-master01 pod]# curl 172.16.58.213
<!DOCTYPE html>
...省略输出 ...
```



#### 9.2 配置livenessProbe（存活探针）
创建一个livenessProbe（存活）探针的 Pod，其中检测8080端口就是为了模拟检测不到端口。也就是10秒时开始检测，超时2秒还没有检测则检测失败一次。第二次检测在第13秒，超时2秒还没有检测则检测失败两次，在第15秒开始Kill容器（此处删除快慢受节点影响，只有删除成功后才会重启Pod中的容器），**如果restartPolicy为Always，则会不断的重启Pod。**

```yaml
[root@k8s-master01 pod]# cat /yaml/pod/livenessProbe.yaml
apiVersion: v1 # 必选，API 的版本号
kind: Pod # 必选，类型 Pod
metadata: # 必选，元数据
    name: nginx-liveness # 必选，符合 RFC 1035 规范的 Pod 名称
spec: # 必选，用于定义 Pod 的详细信息
    containers: # 必选，容器列表
    - name: nginx # 必选，符合 RFC 1035 规范的容器名称
      image: nginx:1.15.12 # 必选，容器所用的镜像的地址
      imagePullPolicy: IfNotPresent
      command: # 可选，容器启动执行的命令
      - sh
      - -c
      - sleep 10; nginx -g "daemon off;"

      livenessProbe: # 可选，健康检查
        tcpSocket: # 端口检测方式
          port: 8080
        initialDelaySeconds: 10 # 初始化时间
        timeoutSeconds: 2 # 超时时间
        periodSeconds: 3 # 检测间隔
        successThreshold: 1 # 检查成功为1次表示就绪
        failureThreshold: 2 # 检测失败2次表示未就绪

      ports: # 可选，容器需要暴露的端口号列表
      - containerPort: 80 # 端口号

    restartPolicy: Never  #重启策略，无论如何都不会重启
```



实验如下：

<img src="https://tupian-1251747426.cos.ap-guangzhou.myqcloud.com/image-20230718234507295.png" width="1178" alt="" title="" crop="0,0,1,1" id="hU6QG" class="ne-image">



#### 9.3 配置readinessProbe（就绪探针）
创建一个readinessProbe（就绪）探针的 Pod，这个就绪探针将会使用HTTP GET请求，访问容器的`/index.html`路径，并且通过容器内的80端口进行连接。如果探针成功连接并返回合适的响应码，那么Pod将被标记为就绪状态，可以开始接收流量。反之，如果探针失败，则Pod将被标记为不可用，不会接收到流量。

```yaml
[root@k8s-master01 pod]# cat /yaml/pod/readinessProbe.yaml
apiVersion: v1 # 必选，API 的版本号
kind: Pod # 必选，类型 Pod
metadata: # 必选，元数据
    name: nginx-readiness # 必选，符合 RFC 1035 规范的 Pod 名称
spec: # 必选，用于定义 Pod 的详细信息
    containers: # 必选，容器列表
    - name: nginx # 必选，符合 RFC 1035 规范的容器名称
      image: nginx:1.15.12 # 必选，容器所用的镜像的地址
      imagePullPolicy: IfNotPresent
      command: # 可选，容器启动执行的命令
      - sh
      - -c
      - sleep 10; nginx -g "daemon off;"

      readinessProbe: # 可选，健康检查。注意三种检查方式同时只能使用一种。
        httpGet: # 接口检测方式
            path: /index.html # 检查路径
            port: 80
            scheme: HTTP # HTTP or HTTPS
            #httpHeaders: # 可选, 检查的请求头
            #- name: end-user
            # value: Jason
        initialDelaySeconds: 10 # 初始化时间, 健康检查延迟执行时间
        timeoutSeconds: 2 # 超时时间
        periodSeconds: 3 # 检测间隔
        successThreshold: 1 # 检查成功为 1 次表示就绪
        failureThreshold: 2 # 检测失败 2 次表示未就绪

      ports: # 可选，容器需要暴露的端口号列表
      - containerPort: 80 # 端口号
    restartPolicy: Never  #重启策略，无论如何都不会重启
```



实验如下：

创建Pod，然后大致13s的时候才READY状态 1/1，因为检测到存活才会READY状态 1/1。前面的时间 READY 0/1 是因为容器中执行sleep 10，等到执行nginx -g "daemon off;" 也有耗时。

<img src="https://tupian-1251747426.cos.ap-guangzhou.myqcloud.com/image-20230719002700137.png" width="1155" alt="" title="" crop="0,0,1,1" id="I7sFe" class="ne-image">



容器中index.html被改名，探针探测不到，READY状态 0/1。则Pod将被标记为不可用，不会接收到流量，同时查看Pod的详细信息。

<img src="https://tupian-1251747426.cos.ap-guangzhou.myqcloud.com/image-20230719002929744.png" width="973" alt="" title="" crop="0,0,1,1" id="wQdsy" class="ne-image">

<img src="https://tupian-1251747426.cos.ap-guangzhou.myqcloud.com/image-20230719003101867.png" width="1225" alt="" title="" crop="0,0,1,1" id="hkd2L" class="ne-image">



将容器中的index.html重命名回来后，READY状态 1/1了。

<img src="https://tupian-1251747426.cos.ap-guangzhou.myqcloud.com/image-20230719003648294.png" width="484" alt="" title="" crop="0,0,1,1" id="ESA2C" class="ne-image">



#### 9.4 配置startupProbe（启动探针）
有时候会有一些应用在启动时需要比较长的初始化时间，在这种情况下，您不想杀死应用程序，也不想对外提供服务。那么之前两种探针的检测参数设置就得检测相对较长间隔时间，或者较多次数。当Pod发生故障，等待时间过长导致"无法容忍"。

要这种情况下，若要不影响对死锁作出快速响应的探测，设置存活探测参数是要技巧的。技巧就是使用startupProbe（启动探针）来设置启动探测，针对 HTTP 或 TCP 检测，可以通过将 `failureThreshold * periodSeconds` 参数设置为足够长的时间来应对糟糕情况下的启动时间。

```yaml
[root@k8s-master01 pod]# cat /yaml/pod/startupProbe.yaml
apiVersion: v1 # 必选，API 的版本号
kind: Pod # 必选，类型 Pod
metadata: # 必选，元数据
    name: nginx-startup # 必选，符合 RFC 1035 规范的 Pod 名称
spec: # 必选，用于定义 Pod 的详细信息
    containers: # 必选，容器列表
    - name: nginx # 必选，符合 RFC 1035 规范的容器名称
      image: nginx:1.15.12 # 必选，容器所用的镜像的地址
      imagePullPolicy: IfNotPresent
      command: # 可选，容器启动执行的命令
      - sh
      - -c
      - sleep 20; nginx -g "daemon off;"

      startupProbe:
        tcpSocket:
            port: 80
        initialDelaySeconds: 10 # 初始化时间, 健康检查延迟执行时间
        timeoutSeconds: 2 # 超时时间
        periodSeconds: 5 # 检测间隔
        successThreshold: 1 # 检查成功为 1 次表示就绪
        failureThreshold: 3 # 检测失败 3 次表示未就绪

      readinessProbe: # 可选，健康检查。
        httpGet: # 接口检测方式
            path: /index.html # 检查路径
            port: 80
            scheme: HTTP # HTTP or HTTPS
            #httpHeaders: # 可选, 检查的请求头
            #- name: end-user
            # value: Jason
        initialDelaySeconds: 10 # 初始化时间, 健康检查延迟执行时间
        timeoutSeconds: 2 # 超时时间
        periodSeconds: 3 # 检测间隔
        successThreshold: 1 # 检查成功为 1 次表示就绪
        failureThreshold: 2 # 检测失败 2 次表示未就绪

      livenessProbe: # 可选，健康检查
        tcpSocket: # 端口检测方式
          port: 80
        initialDelaySeconds: 10 # 初始化时间
        timeoutSeconds: 2 # 超时时间
        periodSeconds: 3 # 检测间隔
        successThreshold: 1 # 检查成功为1次表示就绪
        failureThreshold: 2 # 检测失败2次表示未就绪

      ports: # 可选，容器需要暴露的端口号列表
      - containerPort: 80 # 端口号
    restartPolicy: Never  #重启策略，无论如何都不会重启
```

> Tips：如果 配置了startupProbe，就会先禁用其他探测，直到它成功为止。
>



### 10. Pod的启动流程
<img src="https://tupian-1251747426.cos.ap-guangzhou.myqcloud.com/image-20230719153159490.png" width="1306" alt="" title="" crop="0,0,1,1" id="wJJsg" class="ne-image">

+ 当我们执行kubectl时，Apiserver收到新建pod指令后，pod因选择某个节点部署，此时处于pending状态。
+ 当pod已经选择好节点后，处于ContainerCreating状态。准备拉取镜像。
+ 当成功拉取镜像后，变成Running状态。
+ 当有初始化容器时，先执行初始化容器里面的操作，然后启动主容器。
+ 当启动完成主容器后，如果配置pod探针，先执行startupProbe探针再执行livenessProbe 和 readinessProbe探针。
+ 当健康检查完成后，由Endpoint添加Pod IP。



### 11. Pod的退出流程
#### 11.1 平滑退出流程详解
<img src="https://tupian-1251747426.cos.ap-guangzhou.myqcloud.com/image-20230719155148099.png" width="1278" alt="" title="" crop="0,0,1,1" id="zQt22" class="ne-image">

1. 当我们执行kubectl时，api-server收到删除pod指令后，api-server会将pod状态改为dead状态（看不到），再变为Terminating状态（表示该Pod正在被删除）。K8S 通知 node 执行 docker stop 命令，docker 会先向容器中 PID 为 1 的进程发送系统信号SIGTERM（终止程序）
2. 正常无流量情况下，在宽限期内（30s）内控制面会将关闭的 Pod 从对应的 EndpointSlice（和 Endpoints）对象中移除。
3. 如果配置了PreStop，当流量没有结束的情况下，会在已有的宽限期再加2s，此后控制面会将关闭的 Pod 从对应的 EndpointSlice（和 Endpoints）对象中移除。
4. 超出终止宽限期限时，kubelet 会触发强制关闭过程。容器运行时会向 Pod 中所有容器内仍在运行的进程发送 SIGKILL 信号。kubelet 触发强制从 API 服务器上删除 Pod 对象的逻辑，并将宽限期设置为 0 （这意味着马上删除）。



#### 11.2 指定参数强制删除
执行强制删除时必须同时指定 --force --grace-period=0，强制删除一个 pod 是从集群状态还有 etcd 里立刻删除这个 pod，只是当 Pod 被强制删除时，api-server**不会等待来自 Pod 所在节点上的 kubelet 的确认信息：pod 已经被终止**。在 API 里 pod 会被立刻删除，在节点上， pod被设置成立刻终止后，在强行杀掉前还会有一个很小的宽限期。



#### 12. Pod的生命周期钩子（Hook）
Pod Hook 是由 kubelet 发起的，当容器中的进程启动前或者容器中的进程终止之前运行，这是包含在容器的生命周期之中。我们可以同时为 Pod 中的所有容器都配置 Hook。

Kubernetes 为我们提供了两种钩子函数：

+ PostStart：这个钩子在容器创建后立即执行。但是，并不能保证钩子将在容器 ENTRYPOINT 之前运行，因为没有参数传递给处理程序。主要用于资源部署、环境准备等。不过需要注意的是如果钩子花费太长时间以至于不能运行或者挂起，容器将不能达到 running 状态。
+ PreStop：这个钩子在容器终止之前立即被调用。它是阻塞的，意味着它是同步的，所以它必须在删除容器的调用发出之前完成。主要用于优雅关闭应用程序、通知其他系统等。如果钩子在执行期间挂起，Pod 阶段将停留在 running 状态并且永不会达到 failed 状态。

> Tips：PostStart 或者 PreStop 钩子失败， 它会杀死容器，所以应该让钩子函数尽可能的轻量。当然有些情况下，长时间运行命令是合理的， 比如在停止容器之前预先保存状态。
>



两种方式来实现上面的钩子函数：

+ Exec - 用于执行一段特定的命令，不过要注意的是该命令消耗的资源会被计入容器。
+ HTTP - 对容器上的特定的端点执行 HTTP 请求。



一个示例yaml文件

```yaml
[root@k8s-master01 pod]# cat /yaml/pod/pod-preStop_postStart.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-prestop-poststart
spec:
  containers:
  - name: pod-prestop-poststart
    image: nginx:1.15.12
    imagePullPolicy: IfNotPresent
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]

    ports:
    - containerPort: 80
  restartPolicy: Never
```



根据上面的yaml进行创建一个Pod，配置了postStart和preStop。在启动容器后查看/usr/share/message内容说明已经执行了postStart。

<img src="https://tupian-1251747426.cos.ap-guangzhou.myqcloud.com/image-20230720154924971.png" width="711" alt="" title="" crop="0,0,1,1" id="MfmcG" class="ne-image">



下面看preStop，这里用sleep作示例，当生产环境大多用于优雅退出。也就是使用程序的关闭命令，安全优雅的关闭应用程序。图中进行了kubectl delete操作后，重新启动一个master01节点的ssh会话，快速kubectl get pod查看此时pod的状态，处于Terminating（表示该Pod正在被删除），因为sleep命令没结束，且在宽限期最大程度32秒内，所以会等这个20s的sleep结束后才会结束Pod。

<img src="https://tupian-1251747426.cos.ap-guangzhou.myqcloud.com/image-20230720155229944.png" width="682" alt="" title="" crop="0,0,1,1" id="mHzCP" class="ne-image">

<img src="https://tupian-1251747426.cos.ap-guangzhou.myqcloud.com/image-20230720155333631.png" width="616" alt="" title="" crop="0,0,1,1" id="q44Ia" class="ne-image">



> 本篇文章内容参考杜宽的[《云原生Kubernetes全栈架构师》](https://edu.51cto.com/course/23845.html)，视频、资料文档等，大家可以多多支持！还有[k8s训练营](https://www.qikqiak.com/k8strain/)、[“我为什么这么菜”知乎博主](https://www.zhihu.com/people/xun-zhao-you-yuan-ren-42)等资料文档，感谢无私奉献！
>
