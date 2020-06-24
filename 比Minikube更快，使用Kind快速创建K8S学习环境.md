# 简述

K8S 如火如荼的发展着，越来越多人想学习和了解 K8S，但是由于 K8S 的入门曲线较高很多人望而却步。

然而随着 K8S 生态的蓬勃发展，社区也呈现了越来越多的部署方案，光针对生产可用的环境就有好几种部署方案，对于用来测试和学习环境也同样提供了好几种简单可用的方案。

今天我们来介绍一种用于测试、学习环境快速搭建 K8S 环境的方案：Kind。
  
Kind 的官网是：https://kind.sigs.k8s.io/  

## 那么 Kind 相比于 Minikube 有什么优势呢？

### 基于 Docker 而不是虚拟化

运行架构图如下：
![](https://d33wubrfki0l68.cloudfront.net/79bdd6c59934dec77bf525514c2416f547407720/a470d/docs/images/diagram.png)
Kind 不是打包一个虚拟化镜像，而是直接讲 K8S 组件运行在 Docker。带来了什么好处呢？

1. 不需要运行 GuestOS 占用资源更低。
2. 不基于虚拟化技术，可以在 VM 中使用。
3. 文件更小，更利于移植。

### 支持多节点 K8S 集群和 HA

Kind 支持多角色的节点部署，你可以通过配置文件控制你需要几个 Master 节点，节点 Worker 节点，更好的模拟生产中的实际环境。

# 安装 Kind

Kind 的安装非常简单，只有一个二进制文件，如果大家嫌麻烦，可以直接去 GitHub releases 上下载二进制文件即可。

下面的安装方式来自 Kind 文档 https://kind.sigs.k8s.io/docs/user/quick-start/

## macOS / Linux

```sh
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.8.1/kind-$(uname)-amd64
chmod +x ./kind
mv ./kind /some-dir-in-your-PATH/kind
```

## macOS / Linux 使用 Homebrew

```sh
brew install kind
```

## Windows

```cmd
curl.exe -Lo kind-windows-amd64.exe https://kind.sigs.k8s.io/dl/v0.8.1/kind-windows-amd64
Move-Item .\kind-windows-amd64.exe c:\some-dir-in-your-PATH\kind.exe
```

## Windows 使用 Chocolatey

```cmd
choco install kind
```

# 创建 K8S 集群

> 如果你在 macOS 或 Windows 中使用 Docker 那么至少需要设置 Docker VM 的内存至 6GB，Kind 建议设置为 8GB。  
> 不是不基于虚拟化技术吗？为什么还有 Docker VM？  
> 因为 Docker 其实只支持 Linux，macOS 和 Windwos 是基于虚拟化技术创建了一个 Linux VM。在 Linux 系统上则不存在这些问题。

最简单的情况，我们使用一条命令就能创建出一个单节点的 K8S 环境

```sh
kind create cluster
```

可是呢，默认配置有几个限制大多数情况是不满足实际需要的，默认配置的主要限制如下：

1. APIServer 只监听了 127.0.0.1，也就意味着在 Kind 的本机环境之外无法访问 APIServer
2. 由于国内的网络情况关系，Docker Hub 镜像站经常无法访问或超时，会导致无法拉取镜像或拉取镜像非常的慢

这边提供一个配置文件来解除上诉的限制：

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: "<API_SERVER_ADDRESS>"
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
    endpoint = ["http://f1361db2.m.daocloud.io"]
```

`API_SERVER_ADDRESS` 配置局域网 IP 或想监听的 IP  
`http://f1361db2.m.daocloud.io` 配置 Docker Hub 加速镜像站点

更多的配置（多节点，节点中运行的 K8S 组件版本，APIServer 监听端口，Pod、Service 子网，kubeProxy 模式，端口映射，本地卷持久化）可以查看 Kind 的文档  
https://kind.sigs.k8s.io/docs/user/configuration/

创建完成效果如下：
![](https://cdn.jsdelivr.net/gh/majian159/blogs@master/images/2020_06_24_16_27_M8rO30%20.png)

如果长时间卡在 `Ensuring node image (kindest/node:v1.18.2)` 这个步骤，可以使用 `docker pull kindest/node:v1.18.2` 来得到镜像拉取进度条。

## 复制集群配置文件

Kind 创建集群完成后会将集群的访问配置写入到 `~/.kube/config` 中，可以复制或加入到有 kubectl 工具的环境中。

## 切换 kubectl 集群上下文

```sh
kubectl cluster-info --context kind-kind
```

# 如何访问 K8S 中的 IP

我们在 K8S 中部署应用程序，一般有 4 种方式访问他们。

1. 直接访问 PodIP
2. 通过 Service 的 ClusterIP 访问
3. 通过 Service 的 NodePort 访问
4. 通过 Ingress Service NodePort 访问

其中方式 1、2 需要访问客户端在 K8S 网络环境内。方式 3、4 其实是一种，通过机器的端口映射来触达应用。

个人觉得直接访问 IP+端口更为方便，这边不对 Ingress 做过多的介绍，大家可以看 Kind 关于 Ingress 的文档。  
https://kind.sigs.k8s.io/docs/user/ingress/

这边介绍通过 `kubectl port-forward` 端口转发的方式访问 K8S 中的应用。

## 部署一个 Nginx Deployment 和 Service

yaml 如下：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  ports:
  - name: 80-tcp
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: ClusterIP
```

```sh
kubectl create nginx.yaml
kubectl port-forward service/nginx 8080:80
```

效果如下
![](https://cdn.jsdelivr.net/gh/majian159/blogs@master/images/2020_06_24_16_25_CCABrD%20.jpg)

![](https://cdn.jsdelivr.net/gh/majian159/blogs@master/images/2020_06_24_16_26_jlWNtn%20.png)

可以看到我们将本地的 8080 转发到了 nginx service 的 80 端口，这时访问本地的 8080 端口就可以访问到 service nginx 的 80 端口。

# 常见问题

## Kind 能在一台机器上创建多个 K8S 集群吗？

可以的，`kind create cluster` 提供了 `--name` 参数，可以为 K8S 集群设置名称。  
但是要注意 API Server 的监听地址/端口不能重复或被占用。

## 怎么设置指定的 K8S 版本？

`kind create cluster` 提供了 `--image` 参数，可以设置 `kindest/node` 镜像的版本，一般与 K8S 发布的版本一一对应，具体提供了哪些版本可以去 DockerHub 上查看。  
https://hub.docker.com/r/kindest/node/tags

> 这个功能很酷，在做兼容性测试的时候可以创建一个目标版本的集群进行测试，真是不要太方便。

## 我的应用镜像没有发布到镜像库如何在 K8S 中使用？

可以通过如下几种方式：

1. kind load
2. 本地镜像库
3. 私有镜像库

一般来说可以通过 kind load 将客户机上的镜像加载到 K8S 环境中去。例如将本机的 nginx 镜像加载到 Kind 的 K8S 环境中。

```sh
kind load docker-image nginx nginx
```

甚至可以为镜像起别名

```sh
kind load docker-image nginx nginx:test
```

具体使用方式可以访问 cli 的帮助

```sh
kind load -h
kind load docker-image -h
kind load image-archive -h
```

Kind 的本地镜像库使用方式见文档：https://kind.sigs.k8s.io/docs/user/local-registry/  
私有镜像库使用方式见文档：https://kind.sigs.k8s.io/docs/user/private-registries/

## 还有其它问题？

还有遇到其它问题，欢迎给我留言。
