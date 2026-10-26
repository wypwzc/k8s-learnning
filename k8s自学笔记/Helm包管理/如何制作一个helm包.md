制作一个 Helm Chart 包非常简单，Helm 提供了一个内置命令来帮你自动生成标准的目录结构。以下是制作和使用一个自定义 Chart 的完整步骤：

### 第一步：创建一个新的 Chart 骨架

在终端执行以下命令，创建一个名为 `my-chart` 的模板目录：

Bash

```
helm create my-chart
```

执行后，会生成一个名为 `my-chart` 的文件夹，其标准目录结构如下：

Plaintext

```
my-chart/
  ├── Chart.yaml          # Chart 的基本信息（名称、版本、描述等）
  ├── values.yaml         # 默认的全局配置变量
  ├── charts/             # 依赖的其他子 Chart（通常为空）
  └── templates/          # Kubernetes 资源 YAML 模板目录
      ├── deployment.yaml # 部署 Deployment 模板
      ├── service.yaml    # 服务 Service 模板
      ├── ingress.yaml    # 路由 Ingress 模板
      ├── _helpers.tpl    # 模板辅助函数
      └── NOTES.txt       # 安装成功后展示给用户的提示信息
```

### 第二步：修改基本信息 (`Chart.yaml`)

打开 `my-chart/Chart.yaml` 文件，修改你的应用元数据。例如：

YAML

```
apiVersion: v2
name: my-chart
description: A Helm chart for Kubernetes
type: application
version: 0.1.0        # Chart 本身的版本号
appVersion: "1.0.0"   # 你部署的应用的实际版本号（如镜像版本）
```

### 第三步：配置默认变量 (`values.yaml`)

`values.yaml` 用来定义模板中需要用到的参数及其默认值。例如，配置镜像名称、副本数和端口：

YAML

```
replicaCount: 2

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: ClusterIP
  port: 80
```

### 第四步：编写或修改模板文件 (`templates/`)

进入 `templates/` 目录，这里的文件（如 `deployment.yaml`、`service.yaml`）本质上是标准的 Kubernetes YAML 文件，但嵌入了 Go 模板语法（使用 `{{ .Values.xxx }}` 来引用 `values.yaml` 中的变量）。

例如，在 `deployment.yaml` 中，你会看到类似的代码：

YAML

```
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ .Chart.Name }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
```

你可以根据自己的实际需求，添加或修改这些 YAML 文件（例如增加 ConfigMap、PersistentVolumeClaim 等）。

### 第五步：验证与测试 Chart

在安装之前，建议对 Chart 进行语法检查和本地渲染测试，确保没有拼写错误或模板语法问题：

1. **语法检查（Lint）**：

   Bash

   ```
   helm lint my-chart
   ```

2. **预览渲染结果（Dry Run / Template）**： 查看最终生成的 Kubernetes YAML 文件内容是否正确：

   Bash

   ```
   helm install --dry-run --debug my-release ./my-chart
   ```

### 第六步：安装与打包

1. **直接本地安装**： 如果测试没问题，可以直接将当前目录安装到 Kubernetes 集群中：

   Bash

   ```
   helm install my-release ./my-chart
   ```

2. **打包成压缩文件（分发）**： 如果你需要将 Chart 分发给其他人或上传到私有仓库，可以将其打包为 `.tgz` 压缩包：

   Bash

   ```
   helm package my-chart
   ```

   执行后会生成一个 `my-chart-0.1.0.tgz` 文件，这个文件就是最终的 Chart 包。