# 运维必备 9 个 kubectl 命令

`kubectl` 是 Kubernetes 的命令行管理工具，可用于 Kubernetes 上的应用部署和日常管理。本文列举了 9 个常见的 `kubectl` 命令，并总结了一些实用技巧，帮助系统管理员简化集群管理工作。

---

## 一、使用 Kubectl 查询、创建、编辑和删除资源

对于刚开始使用命令行工具的开发者，最保险的方法是提出问题（读取操作），而不是发出命令（写入操作），因此从使用 `get` 命令开始是个不错的选择。

### 1. Kubectl get

使用 `get` 命令可以获取当前集群中可用的资源列表，包括：
* Namespace
* Pod
* Node
* Deployment
* Service
* ReplicaSet

每个 `get` 命令都能提供集群中可用资源的详细信息。例如 `kubectl get nodes` 命令就提供了 Kubernetes 的状态和版本。

~~~bash
kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
minikube   Ready    master   9d    v1.18.0
~~~

这些命令大多数还具有简写版本。例如，要获取命名空间，可以使用`kubectl get namespaces` 或者 `kubectl get ns` 命令:
```bash
kubectl get ns
NAME              STATUS   AGE
charts            Active   8d
default           Active   9d
kube-node-lease   Active   9d
kube-public       Active   9d
kube-system       Active   9d
```

每个 `get` 命令都可以使用 `--namespace` 或 `-n` 参数指定对应的命名空间。这点对于查看 `kube-system` 中的 Pods 非常有用，因为这些 Pods 是 Kubernetes 自身运行所需的服务。

~~~
$ kubectl get pods -n kube-system
NAME                               READY   STATUS    RESTARTS   AGE
coredns-66bff467f8-mjptx           1/1     Running   2          9d
coredns-66bff467f8-t2xcz           1/1     Running   2          9d
etcd-minikube                      1/1     Running   1          9d
kube-apiserver-minikube            1/1     Running   1          9d
kube-controller-manager-minikube   1/1     Running   2          9d
kube-proxy-rpc9d                   1/1     Running   1          9d
kube-scheduler-minikube            1/1     Running   2          9d
storage-provisioner                1/1     Running   1          9d
~~~

---

### 2. Kubectl create

可以查询资源后，下一步是创建资源。可以用 `kubectl` 在集群中创建任何类型的资源，包括：
* Service
* Cronjob
* Deployment
* Job
* Namespace（ns）

其中，一些资源的创建需要设置配置文件、命名空间以及资源名称。例如，创建命名空间就需要一个额外参数来指定命名空间。

~~~
kubectl create ns hello-there
namespace/hello-there created
~~~

Linux 里可以使用 `cron` 创建定时运行的任务。同样的，这里使用 `cronjob` 每五秒钟返回一次“hello”。

~~~bash
kubectl create cronjob my-cron --image=busybox --schedule="*/5 * * * *" -- echo hello
cronjob.batch/my-namespaced-cron created
~~~

也可以使用 `cronjob` 的简写版本 `cj`。

```
kubectl create cj my-existing-cron --image=busybox --schedule="*/15 * * * *" -- echo hello
cronjob.batch/my-existing-cron created
```

---

### 3. Kubectl edit

当创建好资源后，如果需要修改，就需要使用 `kubectl edit` 命令。

可以用这个命令编辑集群中的任何资源，它会打开默认文本编辑器。如果要编辑现有的 cron job，可以执行：

```bash
kubectl edit cronjob/my-existing-cron
```

**要编辑的配置如下：**

~~~yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  creationTimestamp: "2020-04-19T16:06:06Z"
  managedFields:
  - apiVersion: batch/v1beta1
    fieldsType: FieldsV1
    fieldsV1:
      f:spec:
        f:concurrencyPolicy: {}
        f:failedJobsHistoryLimit: {}
        f:jobTemplate:
          f:metadata:
            f:name: {}
          f:spec:
            f:template:
              f:spec:
                f:containers:
                  k:{"name":"my-new-cron"}:
                    .: {}
                    f:command: {}
                    f:image: {}
                    f:imagePullPolicy: {}
~~~

原本调度时间间隔设置15秒：

![img](https://cdn.nlark.com/yuque/0/2021/png/396745/1620308411613-2ebdc0fb-0bab-43cd-a7a6-1d5e66939c0d.png)

将其改为每25秒：

![img](https://cdn.nlark.com/yuque/0/2021/png/396745/1620308430467-551b94cc-5544-495a-b341-81181cbefde4.png)

修改完成后保存退出，即可看到配置已生效。

~~~bash
kubectl edit cronjob/my-existing-croncronjob.batch/my-existing-cron edited
~~~

另外可以通过 KUBE_EDITOR 命令来使用其他编辑器

~~~bash
$ KUBE_EDITOR="nano" kubectl edit cronjob/my-existing-cron
~~~

---

### 4. Kubectl delete

学会以上命令后，可以进行删除操作。例如删除不再需要的 cronjob 资源：

```bash
kubectl delete cronjob <cronjob-name>
```

> **注意**：如果不知道资源是否有关联信息，最好不要随意删除。删除后无法恢复，只能重新创建。

---

### 5. Kubectl apply

某些命令需要配置文件，而 `apply` 命令可以在集群内将配置文件应用于资源。虽然也可以通过命令行 standard in (STDIN) 来完成，但 `apply` 命令更好一些，因为它可以让您明确知道如何使用集群以及要应用哪种配置文件。

例如，将 Helm 的基于角色的访问控制（RBAC）配置应用于服务帐户：

```bash
kubectl apply -f rbac.yaml
```

> **提示**：可以应用几乎任何配置，但一定要明确所要应用的配置内容，否则可能会引发意料之外的后果。

---

## 二、使用 Kubectl 对 Kubernetes 进行故障排除

### 6. Kubectl describe

`describe` 命令可以查看资源的详细信息。比较常见的用法是查看一个 Pod 或节点信息，以检查是否有异常、资源是否耗尽。

该命令可以查看的资源包括：
* Nodes
* Pods
* Services
* Deployments
* Replica sets
* Cronjobs

**示例：** 查看集群中 `cronjob` 的详细信息：

```bash
kubectl describe cronjob <cronjob-name>
```

---

### 7. Kubectl logs

虽然 `describe` 命令可以让你知道 Pod 内部应用程序发生的事，但 `logs` 命令可以提供 Kubernetes 中 Pod 的更多详细日志。了解这种区别可以帮助开发者更好地针对应用程序内部以及 Kubernetes 内部发生的问题进行故障排除。

**查看 Pod 日志输出示例：**

```text
172.17.0.1 - - [19/Apr/2020:16:01:15 +0000] "GET / HTTP/1.1" 200 612 "-" "kube-probe/1.18" "-"
172.17.0.1 - - [19/Apr/2020:16:01:20 +0000] "GET / HTTP/1.1" 200 612 "-" "kube-probe/1.18" "-"
172.17.0.1 - - [19/Apr/2020:16:01:25 +0000] "GET / HTTP/1.1" 200 612 "-" "kube-probe/1.18" "-"
```

可以使用 `grep` 命令过滤无关信息或查看特定事件。例如过滤掉 `kube-probe`探针日志：

```bash
$ kubectl logs cherry-chart-88d49478c-dmcfv -n charts | grep -vie kube-probe
127.0.0.1 - - [10/Apr/2020:23:01:55 +0000] "GET / HTTP/1.1" 200 612 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:75.0) Gecko/20100101 Firefox/75.0" "-"
```

> **多容器场景**：在包含多个容器的 Pod 中，可以在 `logs` 命令中使用 `-c <容器名称>` 参数来查找指定容器的日志。

---

### 8. Kubectl exec

与 `docker exec` 命令相似，`exec` 命令可以直接进入容器中对应用程序进行故障排除。当 Pod 的日志无法定位问题时特别有用。

> **注意**：使用 `exec` 命令时，必须要以 Pod 内可用的 shell（如 `/bin/bash` 或 `/bin/sh`）作为命令的最后一个参数。

```bash
$ kubectl exec -it cherry-chart-88d49478c-dmcfv -n charts -- /bin/bash
root@cherry-chart-88d49478c-dmcfv:/#
```

---

### 9. Kubectl cp

`kubectl cp` 命令与 Linux `cp` 命令类似，用于容器与本地主机之间复制文件和目录。此外，该命令还能在自动化失败等紧急情况下进行恢复与备份。

**将本地文件拷贝到容器：**
* 命令格式：`kubectl cp <filename> <namespace/podname:/path/to/file>`

```bash
$ kubectl cp commands_copy.txt charts/cherry-chart-88d49478c-dmcfv:commands.txt
$ kubectl exec -it cherry-chart-88d49478c-dmcfv -n charts -- /bin/bash
root@cherry-chart-88d49478c-dmcfv:/# ls
bin  boot  commands.txt  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

**将容器内的文件拷贝到本地计算机：**
* 命令格式：`kubectl cp <namespace/podname:/path/to/file> <local_filename>`

```bash
$ kubectl cp charts/cherry-chart-88d49478c-dmcfv:commands.txt commands_copy.txt
$ ls
commands_copy.txt