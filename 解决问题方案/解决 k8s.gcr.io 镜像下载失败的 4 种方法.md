**解决 k8s.gcr.io 镜像下载失败的 4 种方法**

Kubernetes

目前常用的 Docker Registry 公开服务有：

- docker.io ：Docker Hub 官方镜像仓库，也是 Docker 默认的仓库
- gcr.io、k8s.gcr.io ：谷歌镜像仓库
- quay.io ：Red Hat 镜像仓库
- ghcr.io ：GitHub 镜像仓库

当使用 docker pull 仓库地址/用户名/仓库名:标签 时，会前往对应的仓库地址拉取镜像，标签无声明时默认为 latest， 仓库地址无声明时默认为 docker.io 。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/396745/1651022662481-c4fa5736-b89f-415c-bd2f-5b3701a3a6f4.jpeg)

众所周知的原因，在国内访问这些服务异常的慢，甚至 gcr.io 和 quay.io 根本无法访问。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/396745/1651022662435-05d08c93-177a-44ce-928b-9a3f15b4252b.jpeg)

在学习、研究 K8S 的过程中，会经常遇到镜像拉取不了的网络问题，这并不是镜像本身的问题，而是国内的“国情”导致无法正常访问墙外资源。

这些镜像有的是 K8S 团队自研的插件，也有一些是爱好者开发的第三方组件，正常来说，他们会存放于 gcr.io 或者 quay.io 中。gcr.io 是 谷歌的镜像仓库，是禁止访问的，而 quay.io 是 RedHat 的镜像仓库，可以访问，但速度较慢。

那如何应对这种网络问题呢？

## 解决方案：镜像加速器

针对 Docker Hub ，Docker 官方和国内各大云服务商均提供了 Docker 镜像加速服务。

只需要简单配置一下（以 Linux 为例）：

```bash
sudo mkdir -p /etc/docker

sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["镜像加速器"]
}
EOF

sudo systemctl daemon-reload
sudo service docker restart
```

便可以通过访问国内镜像加速器来加速 Docker Hub 的镜像下载。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/396745/1651022759073-59bba17a-1a19-4aa6-805c-99b0ff742062.jpeg)

不过这种办法也只能针对 docker.io ，其它的仓库地址并没有真正实际可用的加速器（至少目前没找到）。

## 解决方案：用魔法打败魔法

若使用一台魔法机器从 gcr.io 或 quay.io 等仓库先把无法下载的镜像拉取下来，然后重新上传到 docker.io ，是不是就可以使用 Docker Hub 的镜像加速器来下载了。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/396745/1651022758980-bb7d04ba-b793-4fef-aae8-a3463780822c.jpeg)

镜像仓库迁移的功能，这里采用了 Go Docker SDK ，整体实现也比较简单。

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/396745/1651022759108-06198c79-e05b-4211-b5c8-cdd4acb9116c.jpeg)

以需要转换的 gcr.io/google-samples/microservices-demo/emailservice:v0.3.5 为例，使用方式：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/396745/1651022759136-61abc098-e167-48be-a584-3d623b02f9a1.jpeg)

功能实现了，剩下的就是找台带有魔法的机器了。

GitHub Actions 就是个好选择，可以利用提交 issues 来触发镜像仓库迁移的功能。

workflow 的实现如下：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/396745/1651022759088-dd7bcb50-4e55-481a-be51-461bbae101d8.jpeg)

实际的使用效果：

![img](https://cdn.nlark.com/yuque/0/2022/jpeg/396745/1651022759469-16721ea5-beb2-4e2f-b6ac-3ed0e5899f6e.jpeg)

只要执行最终输出的命令，就可以飞快的使用 Docker Hub 的加速器下载 gcr.io 或 quay.io 等镜像了。

## 1、现成的镜像代理仓库

### k8s.gcr.io

这是 gcr.io/google-containers 的仓库，使用阿里云镜像

```bash
docker pull k8s.gcr.io/sig-storage/csi-node-driver-registrar:v2.3.0
# 换成
docker pull registry.aliyuncs.com/google_containers/csi-node-driver-registrar:v2.3.0
```

也可以使用 lank8s.cn，他们的对应关系 k8s.gcr.io --> lank8s.cn，gcr.io --> gcr.lank8s.cn

```bash
docker pull k8s.gcr.io/sig-storage/csi-node-driver-registrar:v2.3.0
# 换成
docker pull lank8s.cn/sig-storage/csi-node-driver-registrar:v2.3.0
```

### quay.io

这是Red Hat运营的镜像库，虽然没有被墙，但下载还是比较慢，可以使用中科大镜像

```bash
docker image pull quay.io/kubevirt/virt-api:v0.45.0
# 换成
docker pull quay.mirrors.ustc.edu.cn/kubevirt/virt-api:v0.45.0
```

## 2、使用自己的代理下载

若有梯子，并且是 http 和 https 协议的（sock5 的不行），可以先在自己的电脑上使用 docker 中配置代理

```json
{
  "proxies":
  {
    "default":
    {
      "httpProxy": "http://127.0.0.1:1080",
      "httpsProxy": "http://127.0.0.1:1080",
      "noProxy": "*.test.example.com,.example2.com"
    }
  }
}
```

有了代理后，就可以访问外面的镜像，下载完成后，使用 `docker save` 打包成 tar 包，再把 tar 上传到服务器上，最后使用 `docker load` 导入到服务器的镜像仓库中。

```bash
# 导出
docker save <image_name>:<tag>  -o /home/image.tar 

# 导入
docker load < /home/image.tar
```

## 3、使用 Github 仓库

在 Github 上有一个仓库（anjia0532/gcr.io_mirror），提供在线镜像代理服务，只要以固定格式创建issue，就会自动触发脚本去下载镜像，然后再使用对应的仓库名去下载即可

```plain
# origin / 原镜像名称
gcr.io/namespace/{image}:{tag}

# eq / 等同于
anjia0532/namespace.{image}:{tag}

# special / 特别的
k8s.gcr.io/{image}:{tag} <==> gcr.io/google-containers/{image}:{tag} <==> anjia0532/google-containers.{image}:{tag}
```

不过要注意的是，该仓库目前仅支持 gcr.io和k8s.gcr.io 镜像

## 4、使用 docker playground 下载

docker playground 提供一个在线的 docker 运行环境，方便新手上去操作学习。

巧的是，docker playground 服务器也在墙外，因此拉取 gcr 的镜像非常快。

利用这个，可以用来拉取 gcr 的镜像，最后再 push 到私人仓库（比如 docker hub，或者  harbor）

首先登陆 docker playground (https://labs.play-with-docker.com)

没帐号的注册一个，进去后，新增一个实例。

然后在终端上拉取镜像，打新 tag，然后再推送到私人仓库，速度都是非常快的。

![img](https://cdn.nlark.com/yuque/0/2022/png/396745/1646320997133-60ba6f74-68f6-45fc-b2c0-0e8646b3d531.png)

完成后，在其他机器上，就可以直接拉取私人仓库的镜像

![img](https://cdn.nlark.com/yuque/0/2022/png/396745/1646320996365-a4ab7669-5ef3-44ff-aa76-ec3f649f8e4c.png)

以上四种方法，数最后一种最为通用，但操作上稍微有一些麻烦，可以根据自己的情况做一些取舍。