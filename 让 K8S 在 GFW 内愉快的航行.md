# 前言
K8S(kubernetes) 日渐火爆，但由于出自Google，对GFW内的用户及其的不友好。  
**而之前的 `*.azk8s.cn` 全能镜像站，也于 2020年4月2日限制了对 Azure China 之外的 IP访问，无疑又是雪上加霜 (很多生产集群应该开始跳脚了)。**
> https://github.com/Azure/container-service-for-azure-china/issues/58  
> ps: 这么大的事件，居然没有提前公告。。。

今天我们来梳理一下，K8S在GFW内如何愉快的航行。  
首先梳理一下 GFW 内 K8S 需要翻越的几座墙。  
## Linux Source
用于安装 docker、kubelet、kubectl、kubeadm 等软件。
## 容器镜像库
目前常用的K8S镜像库有
1. docker.io (docker hub公共镜像库)
2. gcr.io (Google container registry)
3. k8s.gcr.io (等同于 gcr.io/google-containers)
4. quay.io (Red Hat运营的镜像库)

# GFW应对之法
## Linux Source
Linux的源镜像比较简单,这边推荐阿里的镜像源。  
Docker CE: https://developer.aliyun.com/mirror/docker-ce  
Kubernetes: https://developer.aliyun.com/mirror/kubernetes

## 容器镜像库
镜像库是一个比较难找的资源，由于 `*.azk8s.cn` 的关闭目前 `gcr.io` 还没有可替代资源，如大家有相关资源可以联系我，我会添加到文章上。

### Docker Hub
关于 Docker Hub 国内有比较多的加速镜像源。  
例如：
1. 阿里云镜像加速器 (推荐, 需要注册用户)
   - 注册地址: https://cr.console.aliyun.com/cn/instances/mirrors
2. DaoCloud镜像加速器
   - 加速器地址: https://f1361db2.m.daocloud.io
3. 七牛云镜像加速器
   - 加速器地址: https://reg-mirror.qiniu.com

#### 使用方式
修改Docker的配置，为其添加 registry-mirrors ，需要重启docker。  
配置文件路径位于 `/etc/docker/daemon.json`

> 官方文档： https://docs.docker.com/registry/recipes/mirror/
```json
{
  "registry-mirrors": ["https://f1361db2.m.daocloud.io"],
}
```
```sh
# 重启docker
systemctl daemon-reload && systemctl restart docker
```

#### 说明
如果大家在生产环境使用，推荐优先使用阿里云的镜像加速器，虽然注册麻烦了一些。  
这是我目前用下来较为稳定的加速器 (此处极度怀念 dockerhub.azk8s.cn )。

### quay.io
关于 quay.io 可用源很少，目前有如下镜像站
1. quay-mirror.qiniu.com (七牛云, 推荐, 但没有找到长期支持的声明)
2. quay.mirrors.ustc.edu.cn (中科大, 经常不可用, 不推荐)

#### 使用方式
将镜像中的 `quay.io` 替换为 `quay-mirror.qiniu.com`，例如：
```sh
quay.io/prometheus/node-exporter:v0.18.1
# 替换成如下格式
quay-mirror.qiniu.com/prometheus/node-exporter:v0.18.1
```

#### 说明
**这两个源都不是长期稳定**  
七牛云目前可用, 但没有找到任何官方说明长期支持。  
中科大声明有维护, 但测试后基本呈现不可用状态。

### gcr.io 和 k8s.gcr.io
**先说结果，我没有找到这两个源的通用镜像站。**  
这是最难的一部分，也花费了我很多时间。  
> k8s.gcr.io 是 gcr.io/google-containers 的别名，所以  
> k8s.gcr.io/\<image\>:\<tag\> == gcr.io/google-containers/\<image\>:\<tag\>

目前只有折中方案可以曲线救国，但这在使用上还是造来的不变，没有稳定的镜像同步途径，如果你能FQ那么还好一些，如果不行很多K8S生态中的新兴技术你可能很难体验了 (tekton、knative)等，这种情况下你只能去国内镜像站找别人传上来的副本，如：阿里云第三方镜像、dockerhub等。

目前我找到了如下镜像库：
1. googlecontainersmirror (我自己从 gcr.io 同步到Docker Hub的镜像, 只包含核心的几个镜像和版本, 能保障K8S正常运行)
   - 镜像内容: https://hub.docker.com/u/googlecontainersmirror
2. registry.aliyuncs.com/google_containers (阿里云第三方用户上传的镜像，镜像比较多)

#### 使用方式
将镜像中的 `k8s.gcr.io` 或 `gcr.io/google-containers` 替换为 `registry.aliyuncs.com/google_containers` 或 `googlecontainersmirror`，例如：
##### registry.aliyuncs.com/google_containers
```sh
gcr.io/google-containers/kube-proxy:v1.18.0
# 替换为
registry.aliyuncs.com/google_containers/kube-proxy:v1.18.0

k8s.gcr.io/kube-proxy:v1.18.0
# 替换为
registry.aliyuncs.com/google_containers/kube-proxy:v1.18.0
```
##### googlecontainersmirror
```sh
gcr.io/google-containers/kube-proxy:v1.18.0
# 替换为
googlecontainersmirror/kube-proxy:v1.18.0

k8s.gcr.io/kube-proxy:v1.18.0
# 替换为
googlecontainersmirror/kube-proxy:v1.18.0
```

宣称可以提供镜像的站点 (经测试全部不可用)：
1. gcr.mirrors.ustc.edu.cn (经测试不可用)
2. gcr-mirror.qiniu.com (经测试不可用)

#### 说明
1. 为什么要自己同步镜像而不直接使用现有的镜像库？
   > 因为现有的镜像库我没找到任何官方认证，应该是个人传上去的，我们担心跑在生产的K8S集群遭遇到安全问题。  
   > 对于大家来说都是第三方同步的镜像大家可以自行选择，如果是生产用还是推荐推到自己的镜像库来保障镜像安全。
2. googlecontainersmirror 在Docker Hub上拉取速度会不会很慢？
   > 这边取巧的利用了DockerHub加速器，拉取速度取决于加速器的速度，一般情况下很快。

# 总结

## Linux软件镜像源
Docker CE: https://developer.aliyun.com/mirror/docker-ce  
Kubernetes: https://developer.aliyun.com/mirror/kubernetes

## 容器镜像源 (删除线为不可用)
源                     | 镜像
----------------------|----------------------
Docker Hub            | https://\<user_code\>.mirror.aliyuncs.com, https://f1361db2.m.daocloud.io, https://reg-mirror.qiniu.com, ~~dockerhub.azk8s.cn~~
~~gcr.io~~            | ~~gcr.azk8s.cn~~
k8s.gcr.io            | googlecontainersmirror, registry.aliyuncs.com/google_containers, ~~gcr.azk8s.cn/google-containers~~
quay.io               | quay-mirror.qiniu.com, ~~quay.mirrors.ustc.edu.cn~~, ~~quay.azk8s.cn~~
~~mcr.microsoft.com~~ | ~~mcr.azk8s.cn~~