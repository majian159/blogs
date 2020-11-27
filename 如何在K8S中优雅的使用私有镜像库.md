# 前言
在企业落地 K8S 的过程中，私有镜像库 (专用镜像库) 必不可少，特别是在 Docker Hub 开始对免费用户限流之后, 越发的体现了搭建私有镜像库的重要性。

私有镜像库不但可以加速镜像的拉取还可以避免因特有的"网络问题"导致镜像拉取失败尴尬。

当然部署了私有镜像库之后也需要对镜像库设置一些安全策略，大部分私有镜像库采用 IP访问策略+认证 (非公开项目) 的方式对镜像库进行安全保护。

那么对于含有认证限制的镜像库，在 K8S 中该如何优雅的集成呢？  
下文就总结了在 K8S 中使用私有镜像库的几种情况和方式。

# 在 K8S 中使用私有镜像库
首先要确定私有镜像库的授权使用方式，在针对不同的使用方式选择对应的认证配置。

1. 针对节点 (Node)  
    > 这个应该是企业使用 K8S 时最常用的方式，一般也只要使用这个就够了，并且该方案几乎是使用了私有镜像库之后必不可少的配置，它可以做到:   
    在节点环境中进行一定的配置，不需要在 K8S 中进行其它的配置即可享有具体私有库的权限。  
    该方案对该节点上的所有 Pods 生效，**同时还对非 Pods 镜像生效，例如: kubelet 的 pause 镜像，这个非常关键**。
2. 针对服务账号 (ServiceAccount)、针对命名空间 (Namespace)  
    > 配置了该 ServiceAccount 的 Pod 都享有这个 ServiceAccount 所配置的镜像库认证设置。  
    > 还可以利用 K8S 中 default ServiceAccount 机制，达到对一个具体命名空间中没有特殊设置的所有 Pod 生效。
3. 针对 Pod  
   > 针对具体的 Pod 进行认证配置，该 Pod 就会具有私有库的权限。
   > 
   > *Deployment、DaemonSet、StatefulSet、CronJob、Job 等资源都使用了PodTemplate 最终都会以具体的 Pod 资源体验，所以在 PodTemplate 中配置也算对 Pod 配置。*

# 配置步骤
## 前提条件
1. 一个可用私有镜像库 (可用采用 Harbor 搭建)
2. 私有镜像库的账号和密码 (推荐只给只读权限)
3. CRI 基于 Docker (其它的 CRI 暂没有验证)

## 针对节点 (Node) 配置
1. 编写 Docker 配置文件
2. 将 Docker 配置文件放在指定位置
3. 重启 kubelet

### 编写 Docker 配置文件
首先编写 Docker 的认证配置文件, 格式如下:  
```json
{
  "auths": {
    "<HOST>": {
      "auth": "<BASIC_AUTHORIZATION>"
    }
  }
}
```

`<HOST>` 为私有镜像库的地址, 例如: `hub.docker.com`  
`<BASIC_AUTHORIZATION>` 为 `BASE64(<USERNAME>:<PASSWORD>)`  
例如: `cmVhZGVyOjEyMzQ1Ng==`, 其中账号是: `reader`, 密码是: `123456`  
使用 `:` 拼接后进行 base64

完整的配置文件, 例
```json
{
  "auths": {
    "hub.docker.com": {
      "auth": "cmVhZGVyOjEyMzQ1Ng=="
    },
    "harbor.domain.cn": {
      "auth": "cmVhZGVyOiFAIzQ1Ng=="
    }
  }
}
```

如有多个镜像库在 `auths` 节中进行添加即可。

### 将 Docker 配置文件放在指定位置
推荐放在 kubelet 根目录中, 配置文件需以 `config.json` 命名。  
默认的 kubelet 根目录一般为 `/var/lib/kubelet` (如有修改进行替换即可)  
也就是需要放置在 `/var/lib/kubelet/config.json`。

还可以放在以下位置: 
```sh
{--root-dir:-/var/lib/kubelet}/config.json
{cwd of kubelet}/config.json
${HOME}/.docker/config.json
/.docker/config.json
{--root-dir:-/var/lib/kubelet}/.dockercfg
{cwd of kubelet}/.dockercfg
${HOME}/.dockercfg
/.dockercfg
```

> 参考文档:  
> https://kubernetes.io/docs/concepts/containers/images/#configuring-nodes-to-authenticate-to-a-private-registry

#### 放在 ${HOME} 开头的位置
需要在 kubelet service 环境中配置 HOME 的路径, 不然不会生效, 例如: HOME=/root  
**下面是使用 `kubeadm` 安装的环境中可用的脚本, 如果不是请自行配置**
```sh
echo "HOME=${HOME}" >> /var/lib/kubelet/kubeadm-flags.env
```

### 重启 kubelet
如果 init 不是 systemd，请自行替换服务重启的命令
```sh
systemctl daemon-reload; systemctl restart kubelet
```

## 针对服务账号 (ServiceAccount)、针对命名空间 (Namespace)
1. 创建一个 Docker 注册表机密资源
2. 设置 ServiceAccount 的 imagePullSecrets
3. 将 Pod 的 serviceAccountName 设置为该 ServiceAccount 的名称

### 创建一个 Docker 注册表机密资源

#### 使用 kubectl cli 创建注册表机密资源
```sh
kubectl create secret docker-registry <SECRET_NAME> --docker-server=<DOCKER_REGISTRY_SERVER> --docker-username=<DOCKER_USER> --docker-password=<DOCKER_PASSWORD> -n <NAMESPACE>
```
其中  
`<SECRET_NAME>` 是机密资源的名称, 在编辑 sa 资源的时需要引用  
`<DOCKER_REGISTRY_SERVER>` 是私有镜像库的服务器地址  
`<DOCKER_USER>` 是私有镜像库认证的账号  
`<DOCKER_PASSWORD>` 是私有镜像库认证的密码  
`<NAMESPACE>` 是命名空间名称

示例命令如下:  
```sh
kubectl create secret docker-registry docker-reader-secret --docker-server=harbor.domain.cn --docker-username=reader --docker-password=123456 -n basic
```

#### 使用 yaml 创建注册表机密资源

```yaml
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6eyJET0NLRVJfUkVHSVNUUllfU0VSVkVSIjp7InVzZXJuYW1lIjoiRE9DS0VSX1VTRVIiLCJwYXNzd29yZCI6IkRPQ0tFUl9QQVNTV09SRCIsImF1dGgiOiJSRTlEUzBWU1gxVlRSVkk2UkU5RFMwVlNYMUJCVTFOWFQxSkUifX19
kind: Secret
metadata:
  name: docker-reader-secret 
  namespace: default
type: kubernetes.io/dockerconfigjson
```
`.dockerconfigjson` 是base64之后的字符串, 具体内容参考 *"编写 Docker 配置文件"* 节中的内容
```sh
kubectl apply -f docker-reader-secret.yaml
```

### 设置 ServiceAccount 的 imagePullSecrets
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: service1
  namespace: basic
secrets:
- name: service1-token-mp4qs
imagePullSecrets:
- name: docker-reader-secret
```

### 将资源的 serviceAccountName 设置为该 ServiceAccount 的名称
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
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
        image: nginx:1.14.2
      serviceAccountName: service1
```

### 如何针对命名空间内的所有Pod？
K8S 中有个默认的机制，会在命名空间中创建一个名称为 `default` 的 ServiceAccount (sa) 资源。  
并且在资源没有单独指定 `serviceAccountName` 时, 默认使用 `default` 作为serviceAccountName。  
所以我们只需设置 default ServiceAccount 的 imagePullSecrets 即可对该命名空间中没有特殊指定 `serviceAccountName` 字段的 Pod 生效了。

## 针对 Pod
1. 创建一个 Docker 注册表机密资源
2. 设置 Pod 的 imagePullSecrets

### 创建一个 Docker 注册表机密资源
参考 *"创建一个 Docker 注册表机密资源"* 节中的内容

### 一个具体的 Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: foo
  namespace: awesomeapps
spec:
  containers:
    - name: foo
      image: janedoe/awesomeapp:v1
  imagePullSecrets:
    - name: docker-reader-secret
```

### 针对具有 PodTemplate 内容的资源
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
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
        image: nginx:1.14.2
      imagePullSecrets:
      - name: docker-reader-secret
```

# 最后
如果大家的私有镜像库还没有采用认证，就赶紧行动起来吧！  
血的教训，安全问题刻不容缓。