> “Kubernetes 对外流量治理”

前面你学的：

```
Pod
Deployment
Service
```

本质还是：

```
集群内部通信
```

而 Ingress 开始解决：

```
外部用户怎么访问你的系统
```

这是 K8s 网络里非常核心的一层。

------

# 先理解：

用户访问网站时到底发生了什么？

比如：

```
https://api.example.com/user
```

浏览器访问后，请求会经过：

```
用户浏览器
    ↓
公网IP
    ↓
Nginx / LB
    ↓
转发到后端服务
    ↓
SpringBoot / Go / Node
```

而在 K8s 里：

```
Ingress Controller
就是那个：
Nginx / 网关 / 七层代理
```

------

# 为什么 Service 不够？

你前面学过：

```
Service
```

例如：

```
type: NodePort
```

访问：

```
192.168.1.10:30080
```

问题来了：

------

## 问题1：端口乱飞

你有：

```
用户服务：30080
订单服务：30081
支付服务：30082
```

企业不可能这样。

企业一定是：

```
api.company.com/user
api.company.com/order
```

或者：

```
user.company.com
order.company.com
```

------

## 问题2：Service 不会七层路由

Service 只能：

```
四层转发（TCP/UDP）
```

它不懂：

```
HTTP Path
Host
Cookie
Header
```

它看不懂：

```
/user
/order
```

所以：

```
Service ≈ 四层负载均衡
Ingress ≈ 七层负载均衡
```

------

# 什么叫七层？

OSI：

```
七层：
HTTP
HTTPS
域名
URL
Cookie
Header
```

Ingress 能看懂：

```
Host: api.company.com
Path: /user
```

于是：

```
/user → user-service
/order → order-service
```

这叫：

```
七层路由
```

------

# Ingress 本质

核心一句话：

```
Ingress = 路由规则
Ingress Controller = 真正干活的Nginx
```

很多人会混。

------

# 1. Ingress

只是：

```
规则配置
```

例如：

```
/api -> api-service
/web -> web-service
```

它自己不会转发。

------

# 2. Ingress Controller

真正工作的组件。

最常见：

- NGINX Ingress Controller
- Traefik
- HAProxy

其中最经典：

```
Nginx Ingress Controller
```

它本质上：

```
监听 Kubernetes API
↓
发现 Ingress 规则
↓
自动生成 nginx.conf
↓
热更新 Nginx
```

所以：

```
Ingress Controller 本质就是：
“自动配置的 Nginx 集群”
```

------

# 整体流量图（非常重要）

```
用户浏览器
    ↓
域名解析（DNS）
    ↓
Ingress Controller（Nginx）
    ↓
Ingress规则匹配
    ↓
Service
    ↓
Pod
```

以后你会天天排查这一整条链路。

------

# 一个真实例子

## 需求：

```
/shop → shop-service
/api  → api-service
```

------

# Ingress YAML

```
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: app-ingress

spec:
  ingressClassName: nginx

  rules:
  - host: myapp.com

    http:
      paths:

      - path: /shop
        pathType: Prefix
        backend:
          service:
            name: shop-service
            port:
              number: 80

      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

------

# 它会自动生成类似：

```
server {
    server_name myapp.com;

    location /shop {
        proxy_pass http://shop-service;
    }

    location /api {
        proxy_pass http://api-service;
    }
}
```

所以很多人说：

```
Ingress ≈ 自动化Nginx
```

本质没错。

------

# 你在这个阶段会真正接触：

------

# 1. HTTP

必须懂：

```
GET
POST
Header
Cookie
Status Code
```

因为：

```
Ingress 是 HTTP 网关
```

不懂 HTTP 根本学不会。

------

# 2. 域名

必须懂：

```
DNS解析
A记录
CNAME
```

因为：

```
Ingress 基于 Host 路由
```

例如：

```
api.company.com
```

------

# 3. 反向代理

这是核心中的核心。

你会明白：

```
Nginx为什么叫反向代理
```

本质：

```
用户不知道真实后端
代理帮你转发
```

------

# 4. HTTPS / SSL

企业必须 HTTPS。

所以你会学：

```
TLS
证书
cert-manager
Let's Encrypt
```

以后经典问题：

```
为什么 HTTPS 不通？
为什么证书过期？
为什么 TLS 握手失败？
```

全来了。

------

# 5. 灰度发布

Ingress 是灰度核心。

例如：

```
10%流量 → v2
90%流量 → v1
```

或者：

```
/header带beta → 新版本
```

这就是：

```
金丝雀发布
```

也是面试高频。

------

# 这一阶段的学习顺序（推荐）

## 第一部分：先学HTTP基础

必须会：

```
请求
响应
Header
Cookie
状态码
```

推荐：

```
curl -v
```

疯狂抓包看 HTTP。

------

## 第二部分：学 Nginx

必须亲手写：

```
location
proxy_pass
upstream
```

因为：

```
Ingress 本质就是自动生成 Nginx 配置
```

------

## 第三部分：安装 Ingress Controller

最推荐：

- ingress-nginx

------

## 第四部分：练习路由

练：

```
/path 路由
/host 路由
rewrite
```

------

## 第五部分：HTTPS

练：

```
TLS
自签证书
cert-manager
```

------

# 你真正理解 Ingress 后：

会突然发现：

```
K8s 网络其实是：

iptables/ipvs （四层）
+
Nginx/Envoy （七层）
```

这时你对云原生网络理解就开始进入企业级了。