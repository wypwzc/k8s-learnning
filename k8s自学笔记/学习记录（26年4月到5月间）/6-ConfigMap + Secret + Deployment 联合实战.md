# K8s 全链路 HTTPS 部署实战：ConfigMap + Secret + Deployment + Ingress-Nginx + TLS

## 目标场景
构建一个完整的生产级 Web 服务部署流程，实现：
- 静态页面/配置通过 ConfigMap 管理
- 敏感信息（TLS 证书、数据库密码）通过 Secret 管理
- 应用通过 Deployment 部署
- 外部流量通过 Ingress-Nginx 接入
- 强制 HTTPS 访问，TLS 证书自动挂载

----



### 1. 前置准备
- 确认 Ingress-Nginx Controller 已安装（提供验证命令）
- 准备测试域名和自签名证书（或 Let’s Encrypt 方案对比）

### 2. ConfigMap 配置管理
- 创建包含 index.html 和自定义 nginx.conf 的 ConfigMap
- 说明：为什么用 ConfigMap 而不是直接写镜像里？（面试点：配置与镜像解耦）
- 提供 YAML 和 `kubectl create configmap` 两种创建方式

### 3. Secret 管理
- 创建 TLS Secret（type: kubernetes.io/tls）：从 .crt 和 .key 文件导入
- 创建 Opaque Secret：模拟数据库密码等环境变量注入
- 说明：Secret 的数据在 etcd 中如何存储？（面试点：base64 编码 ≠ 加密，配合 RBAC 和 Encryption at Rest）

### 4. Deployment 部署
- 容器挂载 ConfigMap 到 /usr/share/nginx/html
- 容器挂载 Secret 中的 TLS 证书到 /etc/nginx/ssl（或说明 Ingress 层终止 TLS 的方案差异）
- 环境变量注入 Secret 中的数据库密码
- 配置健康检查：livenessProbe 和 readinessProbe 的区别（面试高频）

### 5. Service 暴露
- ClusterIP 类型即可（因为流量走 Ingress，不需要 NodePort/LoadBalancer）
- 说明 Selector 和 Pod Label 的绑定关系

### 6. Ingress 配置（核心）
- apiVersion: networking.k8s.io/v1
- 配置 tls: 字段，引用 Secret 名称
- 配置 rules: host 匹配
- 注解（annotations）：
  - nginx.ingress.kubernetes.io/ssl-redirect: "true"（强制 HTTPS）
  - nginx.ingress.kubernetes.io/rewrite-target: /
- 说明：Ingress-Nginx 的 TLS 终止 vs Pod 层 TLS 终止的区别

### 7. 验证与排障
- 提供完整的验证命令链：
  - kubectl get pods,svc,ingress
  - 查看 Ingress-Nginx 的日志定位 502/503
  - 测试 HTTPS 证书是否生效（curl -v https://xxx）
- 常见错误排查：
  - ConfigMap 挂载后 403 Forbidden（index.html 缺失）
  - Secret 类型错误导致 TLS 挂载失败
  - Ingress 后端 Service 端口不匹配

### 8. 扩展对比（面试加分）
- ConfigMap vs Secret 的本质区别（只是 type 不同，都是 key-value）
- Ingress-Nginx  vs Nginx Ingress Controller（社区版 vs 云厂商版）
- 自签名证书 vs cert-manager 自动签发
- 如果 Secret 更新，已挂载的证书如何热加载？（需要重启 Pod 或触发滚动更新）

## 输出格式要求
- 每个资源提供完整可运行的 YAML
- 每个 YAML 关键字段添加行内注释
- 提供一键部署脚本顺序（apply 的先后依赖关系）
- 最后附一张"数据流转图"：用户 -&gt; DNS -&gt; Ingress-Nginx -&gt; Service -&gt; Pod -&gt; ConfigMap/Secret







## 1. 前置准备

### 确认 Ingress-Nginx Controller 已安装

在开始之前，必须确保集群中已经部署了 Ingress Controller。

```
# 验证 Ingress-Nginx Controller Pod 是否正常运行
kubectl get pods -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# 验证 IngressClass 是否存在（确保处于 (default) 或有明确名称）
kubectl get ingressclass
```

### 准备测试域名与证书方案

本教程使用测试域名：`app.example.com`。

#### 方案对比：自签名证书 vs Let's Encrypt

- **自签名证书**：适合本地开发或企业内网测试。浏览器会提示“不安全”，需要手动信任。
- **Let's Encrypt**：适合公网生产环境。配合 K8s 的 `cert-manager` 可以实现证书的自动申请、续期（90天有效期）和部署，完全免费且受主流浏览器信任。

#### 生成测试用自签名证书

在本地终端执行以下命令生成自签名证书文件：

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=app.example.com/O=DevOps"
```

## 2. ConfigMap 配置管理

### 面试考点：为什么用 ConfigMap 而不是直接写进镜像？

> **配置与镜像解耦（Decoupling）\**是云原生应用的核心原则。 如果将配置文件硬编码在镜像中，每次修改配置（例如修改网页内容或 Nginx 路由）都需要重新触发 CI/CD 流水线去打包、推送并镜像构建，极其低效。使用 ConfigMap 可以实现\**一次构建，多环境复用（Build Once, Run Anywhere）**，只需在不同的集群（开发、测试、生产）应用不同的 ConfigMap 即可。

### 创建方式一：命令行（快速测试）

```
kubectl create configmap web-html-config --from-file=index.html=./index.html
```

### 创建方式二：YAML 声明式（生产推荐）

保存为 `01-configmap.yaml`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-html-config
  namespace: default
data:
  # 挂载到 Nginx 的静态页面
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>K8s HTTPS Real-World</title></head>
    <body>
      <h1>Hello from Kubernetes!</h1>
      <p>ConfigMap 挂载成功，全链路 HTTPS 已生效。</p>
    </body>
    </html>
```

## Secret 管理

### 创建 TLS Secret

使用前置准备中生成的证书文件创建专门用于 TLS 的 Secret：

Bash

```
kubectl create secret tls web-tls-secret --cert=tls.crt --key=tls.key
```

### 创建 Opaque Secret（模拟数据库密码）

Bash

```
kubectl create secret generic db-secret --from-literal=DB_PASSWORD='ProductionPassword2026'
```

### 面试考点：Secret 在 etcd 中如何存储？安全吗？

> **极其重要**：Kubernetes 默认的 Secret **并不安全**。
>
> 1. **存储机制**：Secret 的数据在 etcd 中默认只是经过了 **Base64 编码**，这是一种明文可逆的编码方式，绝非加密（Encryption）。任何拥有集群读取权限或能访问 etcd 的人都可以轻松解码。
> 2. **安全加固方案**：
>    - **RBAC 限制**：严格限制能够 `get/list` Secret 的权限。
>    - **静态加密（Encryption at Rest）**：在 K8s API Server 中配置加密组件（如使用 KMS 插件或 AES-GCM），确保数据写入 etcd 前被真正加密。
>    - **第三方外置机制**：在更高级的生产场景中，推荐使用 HashiCorp Vault、阿里云 KMS 等，配合 External Secrets Operator 引入集群。

#### 对应的声明式 YAML (`02-secrets.yaml`)：

YAML

```
apiVersion: v1
kind: Secret
metadata:
  name: web-tls-secret
type: kubernetes.io/tls # 严格限制类型为 TLS
data:
  # 注意：声明式中必须填入 Base64 后的字符串（此处为演示简写，实际需替换）
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t...
  tls.key: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0t...
---
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque # 常规键值对类型
data:
  # ProductionPassword2026 的 Base64 编码
  DB_PASSWORD: UHJvZHVjdGlvblBhc3N3b3JkMjAyNg== 
```

## 4. Deployment 部署

保存为 `03-deployment.yaml`。本配置中，TLS 证书的终止交由后续的 Ingress 统一处理（标准的云原生架构），Pod 保持轻量级的 HTTP 监听，但同时演示如何挂载 ConfigMap 和注入环境变量。

YAML

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-deployment
  labels:
    app: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx-web
        image: nginx:1.25-alpine # 生产推荐使用轻量化 alpine 镜像
        ports:
        - containerPort: 80 # 容器暴露的端口
          name: http
        env:
        - name: DATABASE_PASSWORD # 注入环境变量
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_PASSWORD
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html # 覆盖 Nginx 默认静态目录
        # 如果需要强行在 Pod 层做 TLS，可以取消下方注释并将 Secret 挂载为文件
        # - name: tls-volume
        #   mountPath: /etc/nginx/ssl
        
        # --- 面试高频：健康检查 ---
        readinessProbe: # 就绪检查：决定 Pod 是否可以接收外部流量
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe: # 存活检查：决定容器是否需要被重启
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
          
      volumes:
      - name: html-volume
        configMap:
          name: web-html-config # 绑定上面创建的 ConfigMap
      # - name: tls-volume
      #   secret:
      #     secretName: web-tls-secret
```

### 面试高频：LivenessProbe vs ReadinessProbe

- **ReadinessProbe（就绪检查）**：检查应用是否**准备好对外提供服务**。如果失败，Kubernetes 会将该 Pod 从 Service 的 Endpoints 列表中移除，流量**不会**路由到它，但**不会重启 Pod**。常用于等待应用加载大文件、初始化数据库连接等场景。
- **LivenessProbe（存活检查）**：检查应用是否**还活着（未死锁或僵死）**。如果失败，Kubernetes 会**直接杀掉该容器并根据重启策略重建**。如果应用只是加载慢而错配了该检查，会导致 Pod 陷入无休止的重启死循环。

## 5. Service 暴露

保存为 `04-service.yaml`。

YAML

```
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  type: ClusterIP # 仅在集群内部暴露，安全且高效，外部流量由 Ingress 代理进入
  ports:
  - port: 80 # Service 暴露的端口
    targetPort: 80 # 对应 Pod 容器内监听的端口（或名称）
    protocol: TCP
  selector:
    app: web-app # 必须与 Deployment 中的 Pod template.metadata.labels 完全一致
```

## 6. Ingress 配置（核心）

保存为 `05-ingress.yaml`。

YAML

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    # 强制将 HTTP 流量重定向到 HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    # 支持 URL 重写路径（本例为根路径，防错配）
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - app.example.com # 绑定证书的域名
    secretName: web-tls-secret # 引用前面创建的 TLS Secret
  rules:
  - host: app.example.com # 域名路由匹配
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-service # 转发给后端 Service
            port:
              number: 80 # 对应 Service 的 port
```

### 架构思考：Ingress 层 TLS 终止 vs Pod 层 TLS 终止的区别

1. **Ingress 边缘终止（推荐方案）**：证书配置在 Ingress Controller 上。流量到达 Ingress 后解密为 HTTP，随后在 K8s 内部集群网络（Overlay Network）中以明文向 Pod 转发。
   - *优点*：极大降低了后端 Pod 的 CPU 算力损耗；证书管理集中化，无需每个解耦的微服务都去配置证书。
2. **Pod 层端到端 TLS 终止（极高安全需求场景）**：Ingress 仅做四层流量转发（Passthrough）或在七层重新加密（Re-encrypt），解密工作在 Pod 内部由 Nginx 镜像或应用程序完成。
   - *缺点*：每个 Pod 都需要加载并同步证书更新，运维成本极高，且严重消耗 Pod 的计算资源。

## 7. 验证与排障

### 一键部署脚本顺序

请严格按照 K8s 资源的依赖关系顺序应用 YAML 文件：

Bash

```
# 1. 部署配置与密钥层
kubectl apply -f 01-configmap.yaml
kubectl apply -f 02-secrets.yaml # 或者使用上述的 kubectl create secret 命令行

# 2. 部署计算与网络层
kubectl apply -f 03-deployment.yaml
kubectl apply -f 04-service.yaml

# 3. 部署路由层（最后部署，防止后端未就绪引发 503）
kubectl apply -f 05-ingress.yaml
```

### 生产级验证命令链

Bash

```
# 一键检查状态：Pod、Service、Ingress 是否全部就绪
kubectl get pods,svc,ingress -o wide

# 实时查看 Ingress-Nginx 控制器的网关路由日志（定位故障的核心手段）
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=50 -f

# 模拟外部客户端进行 HTTPS 访问测试（-k 参数忽略自签名证书的告警，--resolve 本地强制解析域名到 Ingress 的对外 IP）
# 假设你的 Ingress Controller 对外暴露的外部 IP 是 192.168.36.136
curl -vk https://app.example.com --resolve app.example.com:443:192.168.36.136
```

### 常见错误排查脑图

- **403 Forbidden**：通常是由于 ConfigMap 挂载到 Nginx 的静态目录时，目录下的文件权限不正确，或 `index.html` 名字拼写错误，导致 Nginx 找不到索引文件且禁止列出目录。
- **Secret 类型错误导致 TLS 挂载失败**：创建 Secret 时如果没有加 `--type=kubernetes.io/tls` 或是强行写成 Opaque 类型，Ingress 会无法解析证书中的公私钥对，日志中会报 `SSL_CTX_use_PrivateKey_file failed`。
- **502 / 503 Bad Gateway**：
  - 503：Ingress 找不到对应的 Service，或者 Service 后面没有成功绑定的 Endpoints（检查 Selector 标签是否和 Pod 匹配）。
  - 502：Service 成功转发了，但后端 Pod 容器没有监听该端口（检查 Deployment 的 `containerPort` 是否真的是 80）。

## 8. 扩展对比（面试加分项）

### ConfigMap vs Secret 的本质区别

在 K8s 底层，两者的存储和分发机制高度类似，核心区别在于**设计定位**与**节点侧处理**：

- ConfigMap 用于存储明文、非敏感的配置（如 nginx.conf, env 环境变量）。
- Secret 专门针对敏感数据，虽然在 etcd 里默认只有 Base64 编码，但 K8s 保证了 Secret **只会分发到运行了需要该 Secret 的 Pod 的 Node 节点上**，且在 Node 上是存储在 **tmpfs（内存文件系统）** 中，Pod 销毁后立即在内存擦除，不会落盘，从而降低了宿主机泄露风险。

### Ingress-Nginx vs Nginx Ingress Controller

- **Ingress-Nginx**：由 **Kubernetes 社区**主导维护的项目（代码仓库属于 kubernetes/ingress-nginx），开源免费，生态适配度极高，功能丰富。
- **Nginx Ingress Controller**：由 **F5 / NGINX 官方**公司维护。分为开源版和 Plus 商业版。商业版支持 Nginx 官方的高级动态特性、WAF 和企业级商业支持。

### Secret 更新后，已挂载的证书如何热加载？

当更新了 K8s 中的 Secret 后：

1. **卷挂载（Volume Mount）行为**：K8s 的 kubelet 会定期（默认每分钟内）同步更新 Pod 内挂载的文件。
2. **热加载缺陷**：虽然底层文件更新了，但 Nginx 进程本身并不会自动去读取新的证书文件，它依然在内存中持有旧证书。
3. **解决方案**：
   - *方案 A*：触发 Deployment 的滚动更新：`kubectl rollout restart deployment/web-app-deployment`。这是最稳妥的生产做法。
   - *方案 B*：如果使用的是 Ingress 层终止 TLS，**Ingress-Nginx Controller 内部实现了热加载机制**。你只需要更新 Secret，Ingress Controller 会自动感知并热重新加载 Nginx 内存，**不需要**重启业务 Pod。

## 数据流转与架构图

下面是流量从客户端发起，到最终由 Kubernetes 内部组件处理并结合配置信息响应的完整数据流转：

Plaintext

```
+-----------------------------------------------------------------------------------------+
|                                外部网络 (External Network)                              |
|                                                                                         |
|      [ 用户浏览器 ] -------------------------> [ DNS 服务器 ]                           |
|            |                                        |                                   |
|            | 1. 请求解析 app.example.com            | 返回解析出的外部 IP               |
|            v                                        v                                   |
+------------|----------------------------------------------------------------------------+
             |
             | 2. 发起 HTTPS 请求 (Port 443)
             v
+-----------------------------------------------------------------------------------------+
|                             Kubernetes 集群 (Cluster 边界)                              |
|                                                                                         |
|     [ Ingress-Nginx Controller ]  <--- [ Secret: web-tls-secret ] (读取证书解密 TLS)     |
|            |                                                                            |
|            | 3. 路由匹配后，通过 ClusterIP 将流量转发至集群内部                         |
|            v                                                                            |
|     [ Service: web-app-service ]                                                        |
|            |                                                                            |
|            | 4. 负载均衡分发至具体 Endpoints (Pod 业务内网)                            |
|            +-----------------------+                                                    |
|                                    |                                                    |
|                                    v                                                    |
|                      [ Pod: web-app-deployment-xxxx ]                                   |
|                                    |                                                    |
|                         +----------+----------+                                         |
|                         |  Nginx Container   |                                          |
|                         +----------+----------+                                         |
|                                    |                                                    |
|                 +------------------+------------------+                                 |
|                 |                                     |                                 |
|                 v                                     v                                 |
|     [ ConfigMap: web-html-config ]          [ Secret: db-secret ]                       |
|     (挂载为 /.../html/index.html)            (注入为 DB_PASSWORD 环境变量)               |
+-----------------------------------------------------------------------------------------+
```