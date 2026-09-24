# Kubernetes Secret 深度讲解

你现在要真正理解：

```
为什么 K8s 要有 Secret？
它到底解决了什么问题？
和 ConfigMap 本质区别是什么？
```

这个理解非常重要。

因为：

```
ConfigMap + Secret
=
K8s 配置管理核心
```

几乎所有企业 YAML 都会出现。

------

# 一、先理解“配置”到底是什么

在传统开发里：

很多人喜欢把配置写死：

```
db_password = "123456"
```

或者：

```
mysql:
  password: 123456
```

问题：

```
代码和配置耦合
```

会导致：

- 改配置要重新发版
- 密码容易泄露
- 不同环境不好管理
- Git 容易泄密

所以现代架构：

```
配置与程序分离
```

程序：

```
只负责运行逻辑
```

配置：

```
外部注入
```

K8s 就提供了：

| 资源      | 作用       |
| --------- | ---------- |
| ConfigMap | 管普通配置 |
| Secret    | 管敏感配置 |

------

# 二、ConfigMap 和 Secret 的本质

你可以先把它们理解成：

```
K8s里的“键值数据库”
```

例如：

## ConfigMap

```
data:
  nginx.conf: xxx
  log_level: debug
```

------

## Secret

```
data:
  username: cm9vdA==
  password: MTIzNDU2
```

本质都是：

```
key-value
```

只是：

| 类型      | 是否敏感 |
| --------- | -------- |
| ConfigMap | 不敏感   |
| Secret    | 敏感     |

------

# 三、为什么不能只用 ConfigMap？

因为：

```
配置 ≠ 敏感数据
```

比如：

## 普通配置

```
日志级别
端口
nginx配置
```

这些泄露问题不大。

------

## 敏感配置

```
数据库密码
JWT Token
HTTPS证书
AK/SK
```

一旦泄露：

```
直接安全事故
```

所以 Kubernetes：

```
强制把敏感配置独立成 Secret
```

这是：

```
职责分离
```

思想。

------

# 四、Secret 的真正本质

这是重点。

很多初学者以为：

```
Secret = 加密
```

实际上：

```
不是
```

K8s 默认：

```
只是 Base64 编码
```

例如：

```
123456
```

变成：

```
MTIzNDU2
```

任何人都能解码：

```
echo MTIzNDU2 | base64 -d
```

所以：

```
Secret 的核心意义：
不是加密
而是“敏感资源隔离”
```

即：

```
K8s知道：
这是敏感数据
```

然后：

- 权限单独控制
- RBAC 单独控制
- 审计单独处理
- 可以接入真正加密系统

------

# 五、Secret 与 ConfigMap 最核心区别

# 1. 使用目的不同

| 类型      | 作用        |
| --------- | ----------- |
| ConfigMap | 管配置      |
| Secret    | 管密码/证书 |

------

# 2. 存储方式不同

## ConfigMap

明文：

```
data:
  password: 123456
```

------

## Secret

Base64：

```
data:
  password: MTIzNDU2
```

------

# 3. API 字段不同

## ConfigMap

```
kind: ConfigMap
```

------

## Secret

```
kind: Secret
```

------

# 4. 权限控制不同

企业里：

```
ConfigMap：
很多人可读
```

但：

```
Secret：
严格限制
```

因为：

```
密码不能随便看
```

------

# 六、Secret 生命周期（非常重要）

你要理解：

```
Secret 从创建到 Pod 使用的全过程
```

------

# 第一步：创建 Secret

例如：

```
kubectl create secret generic mysql-secret \
--from-literal=username=root \
--from-literal=password=123456
```

K8s 做了什么？

```
API Server
↓
保存到 etcd
↓
类型是 Secret
```

------

# 第二步：Pod 引用 Secret

Pod YAML：

```
env:
- name: MYSQL_PASSWORD
  valueFrom:
    secretKeyRef:
      name: mysql-secret
      key: password
```

------

# 第三步：Kubelet 拉取 Secret

节点上的 kubelet：

```
监听 Pod
↓
发现引用 Secret
↓
向 API Server 请求 Secret
```

------

# 第四步：注入容器

两种形式：

| 方式   | 本质     |
| ------ | -------- |
| env    | 环境变量 |
| volume | 文件挂载 |

------

# 七、为什么企业更喜欢 Volume 挂载？

因为：

```
环境变量有风险
```

例如：

```
env
```

可能直接看到密码。

甚至：

```
进程崩溃日志
```

也可能打印出来。

所以很多企业：

```
更推荐 Secret Volume
```

------

# 八、Secret Volume 本质

假设：

```
data:
  username: root
  password: 123456
```

挂载后：

```
/etc/secret/username
/etc/secret/password
```

注意：

```
每个 key → 一个文件
```

文件内容：

```
文件内容 = value
```

------

# 九、ConfigMap Volume 与 Secret Volume 本质一致

这是重点。

其实：

```
ConfigMap 和 Secret 用法几乎一样
```

都支持：

| 功能        | ConfigMap | Secret |
| ----------- | --------- | ------ |
| env 注入    | ✓         | ✓      |
| Volume 挂载 | ✓         | ✓      |
| key-value   | ✓         | ✓      |

真正区别只有：

```
“是否敏感”
```

------

# 十、Secret 类型（企业重点）

------

# 1. Opaque（默认）

最常用。

普通 key-value。

```
type: Opaque
```

例如：

```
数据库密码
Redis密码
Token
```

------

# 2. kubernetes.io/tls

HTTPS 证书。

Ingress 必学。

包含：

```
tls.crt
tls.key
```

创建：

```
kubectl create secret tls nginx-tls \
--cert=tls.crt \
--key=tls.key
```

------

# 3. docker-registry

私有仓库认证。

例如：

- Docker Hub 私仓
- Harbor
- 阿里云镜像仓库

------

# 十一、Secret 与 etcd

K8s 所有资源：

```
最终都存 etcd
```

包括：

- Pod
- Deployment
- Secret
- ConfigMap

所以：

```
etcd 被偷
=
所有 Secret 被偷
```

这也是：

```
生产环境必须开启 etcd encryption
```

原因。

------

# 十二、企业真正的 Secret 管理

大型企业很少直接用 K8s Secret。

而是：

```
Vault
↓
动态生成密码
↓
K8s 拉取
```

最经典：

- HashiCorp Vault

------

# 十三、ConfigMap + Secret 最经典组合

企业里：

## ConfigMap

```
MYSQL_HOST=mysql
MYSQL_PORT=3306
```

------

## Secret

```
MYSQL_PASSWORD=123456
```

程序：

```
同时读取：
ConfigMap + Secret
```

这就是：

```
12-factor app
```

思想。

------

# 十四、面试真正会问的东西

------

# 1. Secret 和 ConfigMap 区别？

标准答案：

```
两者本质都是 key-value 配置资源。

ConfigMap 存普通配置。
Secret 存敏感数据。

Secret 默认只是 Base64 编码，不是真加密。
```

------

# 2. Secret 如何注入 Pod？

```
env
volume
```

------

# 3. Secret 安全吗？

```
默认不安全
只是 Base64
```

生产：

```
etcd encryption
Vault
RBAC
```

------

# 4. 为什么企业推荐 Volume？

```
避免环境变量泄露
```

------

# 十五、你现在真正应该理解的架构思想

这是核心中的核心。

K8s：

```
把“程序”和“配置”彻底解耦
```

程序镜像：

```
不可变
```

配置：

```
动态注入
```

Secret/ConfigMap：

```
本质是“配置中心”
```

------

# 十六、你下一步必须做的实战（非常重要）

建议顺序：

## 1. ConfigMap 挂载 nginx.conf

你会理解：

```
配置热更新
```

------

## 2. Secret 注入 MySQL 密码

你会理解：

```
敏感配置管理
```

------

## 3. Ingress TLS Secret

你会真正理解：

```
HTTPS证书
```

------

## 4. ConfigMap + Secret 联合部署 SpringBoot

这时你就真正进入企业玩法了。